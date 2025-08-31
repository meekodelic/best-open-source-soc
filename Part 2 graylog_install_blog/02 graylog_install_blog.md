📊 Part 2: Installing and Configuring Graylog for Log Ingestion
# **Introduction**
In this section of my SOC (Security Operations Center) project, I focused on deploying and configuring \*\*Graylog\*\* on Ubuntu 24.04 to handle \*\*log ingestion\*\*. Since MongoDB packages for Ubuntu 24.04 (Noble) are not officially available yet, I used the \*\*Ubuntu 22.04 (Jammy) repository workaround\*\*. I also ensured Graylog could securely connect to my \*\*Wazuh Indexer\*\* using SSL certificates.\
\
Here’s the \*\*step-by-step breakdown\*\* of my installation and configuration.
# **1) Installing MongoDB 7.0 on Ubuntu 24.04**
Graylog requires MongoDB. Since Noble doesn’t yet have official packages, I installed MongoDB 7.0 from the Jammy repo.

Import the MongoDB 7.0 GPG key:

***curl -fsSL https://www.mongodb.org/static/pgp/server-7.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-7.0.gpg --dearmor***

Add the Jammy MongoDB repo (works on Noble):

***echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-7.0.gpg ] https://repo.mongodb.org/apt/ubuntu jammy/mongodb-org/7.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-7.0.list***

Update and install MongoDB:

***sudo apt update\
sudo apt install -y mongodb-org***

Start and enable MongoDB service:

***sudo systemctl daemon-reload\
sudo systemctl enable mongod.service\
sudo systemctl restart mongod.service\
sudo systemctl status mongod.service***
# **2) Installing Graylog**
With MongoDB running, I installed the Graylog repository and the server itself:

***wget https://packages.graylog2.org/repo/packages/graylog-4.3-repository\_latest.deb\
sudo dpkg -i graylog-4.3-repository\_latest.deb\
sudo apt-get update && sudo apt-get install graylog-server graylog-integrations-plugins***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.001.png)
# **3) Configuring Certificates**
To enable secure communication with Wazuh Indexer, I configured RootCA certificates inside Graylog:

***mkdir /etc/graylog/server/certs\
cp -a /usr/lib/jvm/java-11-openjdk-amd64/lib/security/cacerts /etc/graylog/server/certs/cacerts\
cp ~/wazuh-certificates/root-ca.pem /etc/graylog/server/certs/***

Then I imported the RootCA into Java’s keystore:

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.002.png)***keytool -importcert -keystore /etc/graylog/server/certs/cacerts -storepass changeit -alias root\_ca -file /etc/graylog/server/certs/root-ca.pem***

# **4) Java Environment Check and Copying Wazuh Cert to graylog**
I ensured OpenJDK 11 was installed and properly linked:

***apt update\
apt install openjdk-11-jdk -y\
java -version***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.003.png) 

now

✅ 1. Check where the wazuh certificate file is file really is:

Let’s go home and look:

ls ~/wazuh-certificates/

Look for this file in the list:

root-ca.pem

If it's there, awesome!

✅ 2. Now copy it to Graylog's certs:

cp ~/wazuh-certificates/root-ca.pem /etc/graylog/server/certs/

You should see:

cacerts  root-ca.pem

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.004.png)



keytool -importcert -keystore /etc/graylog/server/certs/cacerts -storepass changeit -alias root\_ca -file /etc/graylog/server/certs/root-ca.pem

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.005.png) 

we have to change directory---

root@pagla:/etc/graylog/server/certs# cd ..

root@pagla:/etc/graylog/server# ls

certs  log4j2.xml  node-id  server.conf

# **5) Configuring Graylog**
Graylog requires two secrets: password\_secret and root\_password\_sha2.

Generate password\_secret:

***pwgen -N 1 -s 96***

Generate root\_password\_sha2:

***echo -n "Enter Password: " && head -1 </dev/stdin | tr -d '\n' | sha256sum | cut -d" " -f1***

Edit /etc/graylog/server/server.conf and add:

***password\_secret = <generated\_secret>\
root\_password\_sha2 = <sha256\_of\_password>***
# **6) Connecting to Wazuh Indexer**

Now we have to Configure the Connection to your Wazuh-Indexer:

1<sup>st</sup> login to wazuh indexer.

Go to security and create an internal user:

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.006.png) 

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.007.png) 

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.008.png)

I created a user in Wazuh Indexer for Graylog (username: graylog) and configured Graylog’s connection:

***elasticsearch\_hosts = https://graylog:pass@172.23.189.96:9200***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.009.png)

Then enabled and started Graylog:

***systemctl daemon-reload\
systemctl enable graylog-server.service\
systemctl start graylog-server.service\
systemctl --type=service --state=active | grep graylog***
# **7) Fixing SSL Trust Issues**
To fix SSL certificate trust issues, I modified Java options for Graylog in /etc/default/graylog-server:

***GRAYLOG\_SERVER\_JAVA\_OPTS="$GRAYLOG\_SERVER\_JAVA\_OPTS -Dlog4j2.formatMsgNoLookups=true -Djavax.net.ssl.trustStore=/etc/graylog/server/certs/cacerts -Djavax.net.ssl.trustStorePassword=changeit"***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.010.png)
# **8) Enabling Graylog Web UI Access**
By default, Graylog binds only to loopback. To allow external access, I changed this line in /etc/graylog/server/server.conf:

***http\_bind\_address = 0.0.0.0:9000***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.011.png)

Then restarted Graylog:

***systemctl restart graylog-server\
netstat -ltpnd***

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.012.png)







![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.013.png)

![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.014.png)

We can see Graylog has created indexes In wazuh.------




![](Aspose.Words.36a35f73-667d-498c-9627-7646b303812b.015.png)
# **✅ Conclusion**
With Graylog installed and connected to Wazuh Indexer:\
\- MongoDB was installed successfully on Ubuntu 24.04 using the Jammy repo.\
\- Graylog was secured with certificates and password protection.\
\- SSL trust issues were resolved by importing Wazuh’s RootCA.\
\- Hostname mismatch was fixed by connecting via IP.\
\- Graylog WebUI is accessible at http://<server-ip>:9000.\
\
This completes the log ingestion layer of my SOC project. In the next step, I’ll integrate Wazuh Manager + Fluent Bit to forward and normalize logs. 🚀
