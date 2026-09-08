# Lab: Basic SSRF against another back-end system

This lab is quite same as before that has a stock check feature which fetches data from an internal system.

We have to scan the internal 192.168.0.X range for an admin interface on port 8080, then use it to delete the user carlos in order to solve the lab.

Alright so i opened my bursuite turned accessed the lab URL and go to any random item and scroll below to see the check stock

![check stock of a random item](Images/2check_stock.png)

Before clicking on the check stock button turn the intercept on in the burpsuite then click on the check stock button. The POST request will have this stockAPI option at the last.

![stockAPI fetch request](Images/2stockAPI_POST_request_fetch.png)

Change it to this

![changing the stockAPI request](Images/2making_changes_in_request.png)

And dont' forward it but instead send it to the intruder to add the payload position on the 1 at the IP Address. Since the request will have 192.168.0.1 in the URL you will capture. Select the Sniper Attack change the payload type to Numbers select it from 1 to 255.

![adding payload position](Images/2adding_payload_position.png)

![payload configuration](Images/2payload_configuration.png)

Start the attack and wait for it to complete you will see on of the numbers have different length.

![IP Address for admin](Images/2length_response.png)

Now change the last number of the IP Address to the number which you just found and forward the request.

![changing IP](Images/2after_changin_IP.png)

When you come back to the page of you will see that you have admin access now we have to write the name down of the user we have to delete in this case it's carlos.

![admin access](Images/2admin_access.png)

Now to delete the user click on the check stock and this change the request to this and the used will be deleted and the lab will be solved.

```
stockApi=http%3a%2f%2f192.168.0.218%3a8080%2fadmin%2fdelete%3fusername%3dcarlos
```
