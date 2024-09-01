# LogStash Keystore for Microsoft Sentinel / Entra ID secrets

Per the implementation in [README](/README.md), LogStash is defined as a systemd service. We will utilize the Environment variable of a systemd service to define the LogStash keystore password, allowing LogStash access to a keystore containing credentials related to the initial implementation.

[Microsoft recommends](https://learn.microsoft.com/en-us/azure/sentinel/connect-logstash-data-connection-rules#:~:text=For%20security%20reasons%2C%20we%20recommend%20that%20you%20don%27t%20implicitly%20state%20the%20client_app_Id%2C%20client_app_secret%2C%20tenant_id%2C%20data_collection_endpoint%2C%20and%20dcr_immutable_id%20attributes%20in%20your%20Logstash%20configuration%20file.) that you enter the following values into the keystore (instead of storing in plaintext on 50-output.conf):
- client_app_Id
- client_app_secret
- tenant_id
- data_collection_endpoint
- dcr_immutable_id

## Setup

1. Edit the systemd configuration. This will create a new file in `/etc/systemd/system/logstash.service.d/override.conf`
```
systemctl edit logstash
```

2. Add the Environment variable for LOGSTASH_KEYSTORE_PASS. Be sure to put a password here. Since this is one and done, I'd make it strong.
```
[Service]
Environment="LOGSTASH_KEYSTORE_PASS=<CHANGE ME>"
```

3. Modify the file permissions of the `/etc/systemd/system/logstash.service.d/` file. This will prevent unauthorized users from accessing the keystore credential.
```
chmod -R 600 /etc/systemd/system/logstash.service.d/
```

4. Create the keystore:
```
set +o history
export LOGSTASH_KEYSTORE_PASS=<CHANGE ME>
set -o history
/usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash create
```

5. Add the keys to the keystore appropriately:
```
/usr/share/logstash/bin/logstash-keystore --path.settings /etc/logstash add <key_1> <key_2> <...>
```

6. Define the keys in 50-output.conf using the newly defined keys.
```
output {
  microsoft-sentinel-log-analytics-logstash-output-plugin {
      client_app_Id => "${<key_1>}"
      client_app_secret => "${<key_2>}"
      <...>
  }
}
```
