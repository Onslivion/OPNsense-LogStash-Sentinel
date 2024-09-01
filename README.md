# OPNsense --> LogStash --> DCR --> Microsoft Sentinel / Log Analytics
This is heavily based on [Truvis Thorton's implementation](https://github.com/Truvis/Sentinel/tree/main), which may have derived from [noodlemctwoodle's pf-azure-sentinel](https://github.com/noodlemctwoodle/pf-azure-sentinel); credits to them for their efforts. 

There were fields I cleaned up a bit or removed to reduce the amount of data sent to Azure. I would recommend reviewing the fields removed (they've been commented out) and see if they apply to your needs (check the 03-filter.conf and 45-prune.conf files for these pruned elements).

Additionally, these implementations utilize the now legacy HTTP Collector APIs from Log Analytics. I've modified 50-output.conf to instead use the DCR-based (Data Collection Rule) APIs that intend to supersede the legacy APIs.

## Prerequisites
- An instance of Debian/Ubuntu (ensure it's up to date via APT).
- A Microsoft Sentinel instance in your desired region.
- (Optional, highly recommended) Configured network security perimeters to restrict ingestion and querying access to specific networks. *This requires preview functionality.* OR, the use of Private Link.

## Installation

### LogStash
1. Install an OpenJDK runtime (preferably headless). At the current time, I have used OpenJDK JRE 17.
```
sudo apt install openjdk-17-jre-headless
```

3. [Install LogStash](https://www.elastic.co/guide/en/logstash/current/installing-logstash.html).
```
wget -qO - https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic-keyring.gpg
sudo apt-get install apt-transport-https
echo "deb [signed-by=/usr/share/keyrings/elastic-keyring.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee -a /etc/apt/sources.list.d/elastic-8.x.list
sudo apt-get update && sudo apt-get install logstash
```

3. Download all configuration files (see conf.d folder in repo) and place them in `/etc/logstash/conf.d/`.
4. Update 20-interfaces.conf to align with the interfaces in your OPNsense instance.
```
if [interface] =~ /^**ovpnc1**$/ <change ** ** to interfaces on your firewall> {
  mutate {
    add_field => { "[interface][alias]" => "OpenVPN-1" <make a simple alias>}
    add_field => { "[network][name]" => "Outgoing Connections" <make a user-readble name/phrase>}
  }
}
```
5. Update the timezone for 03-filter.conf near the top of the file. This will impact readouts for the "[event][created]" field.
6. Install the microsoft-sentinel-log-analytics-logstash-output-plugin.
```
/usr/share/logstash/bin/logstash-plugin install microsoft-sentinel-log-analytics-logstash-output-plugin
```

### Log Analytics Table Configuration
The [microsoft-sentinel-log-analytics-logstash-output-plugin](https://github.com/Azure/Azure-Sentinel/tree/master/DataConnectors/microsoft-sentinel-log-analytics-logstash-output-plugin) uses the DCR-based (Data Collection Rule) APIs. This plugin is currently in public preview. There is also a way to utilize the legacy-based HTTP Collector APIs instead if you prefer (see [Introduction](https://github.com/Onslivion/OPNsense-LogStash-Sentinel#Introduction).

I manually flattened all nested entries using 47-flatten.conf to be most compatible / readable in Azure. 

The following instructions will align mostly with the instructions provided here: [Tutorial: Send data to Azure Monitor Logs with Logs ingestion API (Azure portal)](https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#create-new-table-in-log-analytics-workspace)

0. Create an Entra app registration and a data collection endpoint (although optional, might as well be consistent until Azure updates their documentation and UI).
   Observe the configuration in 50-output.conf. 

1. Initiate LogStash once. This will generate an example file in /tmp/.
```
sudo systemctl start logstash
```

3. Create a new DCR-based table in the Log Analytics workspace per the above guide. Use the json file placed in /tmp/ to generate the table.

4. When prompted to make a TimeGenerated entry, I recommend using the following query. This query will make the LogStash generation timestamp the TimeGenerated field (and remove the now duplicate).

```kql
source
| extend TimeGenerated = todatetime(ls_timestamp)
| project-away ls_timestamp
```

4. Finalize the creation of the table, then navigate to the newly created Data Collection Rule made by the table.

5. Assign the app registration the appropriate permissions [per the guide]("https://learn.microsoft.com/en-us/azure/azure-monitor/logs/tutorial-logs-ingestion-portal#assign-permissions-to-the-dcr"). This change may take a minute to propogate - you may experience 403 errors.
  
6. Take the "Immutable Id" field and paste it into the "dcr_immutable_id" field (50-output.conf).

7. Navigate to "Data sources" under "Configuration". Take the "Data source" field (should be the only one) from the first entry in the table and paste it into the "dcr_stream_name" field (50-output.conf).

8. Navigate to the newly created Data Collection Endpoint, then copy the "Logs Ingestion" field into the "data_collection_endpoint" field (50-output.conf).

9. Gather app ID, app secret, and tenant ID information from the Entra app registration created earlier. Paste them into "client_app_Id", "client_app_secret", and "tenant_id" respectively.

10. Uncomment the fields and comment the sample_file related parameters in 50-output.conf.

11. Restart LogStash.
```
sudo systemctl restart logstash
```

LogStash should now be sending log entries to the data collection endpoint, which will send the logs to a custom table in Sentinel / Log Analytics.

I highly recommend removing the entries from 50-output.conf and entering them into the LogStash Keystore. A brief guide can be found at [keystore.md](https://github.com/Onslivion/OPNsense-LogStash-Sentinel/tree/main/keystore.md).



