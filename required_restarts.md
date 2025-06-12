# This DQL is used to report all the restarts required on an environment to ensure proper monitoring by the OneAgent
# To run this DQL you're goint to be prompted for permissions to run the code as it is using the JavaScript feature of the notebooks or dashboards
# The original code was took from an Slack Channel but Emmanuel Ruiz(emmanuel.ruiz@dynatrace.com) and me (juan.forero@dynatrace.com) did some changes for it to only show required restarts instead of the status of all the processes.
# if you want to see all the processes remove the folllowing lines of code     if(match.state != "ok" && match.state != "deep_monitoring_successful") {return match ? { ...dqlItem, severity: match.severity, state: match.state } : dqlItem;}return null;}).filter(Boolean);

import { monitoredEntitiesMonitoringStateClient } from "@dynatrace-sdk/client-classic-environment-v2";
import { queryExecutionClient } from "@dynatrace-sdk/client-query";
async function fetchAllMonitoringStates(pgID) {
  const pgID_string = `${pgID.map(id => `"${id}"`).join(",")}`;
  let nextPageKey = null;
  const monitoringStates = [];
  do {
    const params = nextPageKey
      ? { nextPageKey }
      : { entitySelector: `entityId(${pgID_string})`};
    const result = await monitoredEntitiesMonitoringStateClient.getStates(params);
    monitoringStates.push(...result.monitoringStates);
    nextPageKey = result.nextPageKey;
  } while (nextPageKey);
  return monitoringStates
}
export default async function () {
  const now = new Date();
  const twentyMinutesAgo = new Date(now.getTime() - 20 * 60 * 1000);
  const formatDate = (date) => date.toISOString();
  const dqlQuery = `
    fetch dt.entity.process_group_instance
    | fieldsAdd host.id=belongs_to[dt.entity.host], pg_id=instance_of[dt.entity.process_group], cgi_id=(belongs_to[dt.entity.container_group_instance])
    | lookup [fetch dt.entity.host], sourceField:host.id, lookupField:id, fields:{host.name=entity.name, host.hostGroupName=hostGroupName}
    | lookup [fetch dt.entity.process_group], sourceField:pg_id, lookupField:id, fields:{process.name=entity.name}
    | lookup [fetch dt.entity.container_group_instance
          | fieldsAdd pod_id=belongs_to[dt.entity.cloud_application_instance], namespace_id=belongs_to[dt.entity.cloud_application_namespace], node_id=belongs_to[dt.entity.host]
          | lookup [fetch dt.entity.cloud_application_instance], sourceField:pod_id, lookupField:id, fields:{pod_name=entity.name}
          | lookup [fetch dt.entity.cloud_application_namespace], sourceField:namespace_id, lookupField:id, fields:{namespace=entity.name}
          | lookup [fetch dt.entity.host], sourceField:node_id, lookupField:id, fields:{node_name=entity.name}], sourceField:cgi_id, lookupField:id, fields:{pod_name, namespace, node_name}
    | fields id, process.name,host.id, host.name, host.hostGroupName, pod_name, namespace, node_name, managementZones
    | sort process.name desc
  `;
  const queryExecution = await queryExecutionClient.queryExecute({ body: { query: dqlQuery } });
  let dqlResults = [];
  while (true) {
    const queryPollResult = await queryExecutionClient.queryPoll({ requestToken: queryExecution.requestToken });
    if (queryPollResult.state !== "RUNNING") {
      dqlResults = queryPollResult.result.records;
      break;
    }
  }
  let pgID = dqlResults.map(item => item.id);
  const processAllChunks = async () => {
    const allMonitoringStates = [];
    for (let i = 0; i < pgID.length; i += 50) {
      const chunk = pgID.slice(i, i + 50);
      const chunkStates = await fetchAllMonitoringStates(chunk);
      allMonitoringStates.push(...chunkStates);
    }
    return allMonitoringStates;
  };
  const monitoringStates = await processAllChunks();
  const mergedResults = dqlResults.map(dqlItem => {
    const match = monitoringStates.find(apiItem => apiItem.entityId === dqlItem.id);
    if(match.state != "ok" && match.state != "deep_monitoring_successful") {
      return match ? { ...dqlItem, severity: match.severity, state: match.state } : dqlItem;
    }
    return null;
  }).filter(Boolean);
  return mergedResults
}
