# Lab: Remote code execution via web shell upload

This lab contains a vulnerable image upload function. It doesn't perform any validation on the files users upload before storing them on the server's filesystem.

To solve the lab, upload a basic PHP web shell and use it to exfiltrate the contents of the file /home/carlos/secret. Submit this secret using the button provided in the lab banner.

You can log in to your own account using the following credentials: wiener:peter



So, first open your bursuite and go to my account page in the lab to login with the given credentials. Then you will see there is a file upload sectioon for the profile picture, upload an arbitary image in it (any image that you want).
