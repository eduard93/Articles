
1. Start IAM as usual.
2. Go to the Admin Portal -> [Teams](http://localhost:8002/teams/#admins).
3. Press `+ Invite Admin` button.
4. Fill all fields, including email and add `super-admin` role:![image](https://user-images.githubusercontent.com/5127457/158452175-7aaada69-2960-4eb4-b21f-847b69586609.png)
5. Press `Invite Admin` button.
6. Back on the Admin list, press `View` button for our new admin: ![image](https://user-images.githubusercontent.com/5127457/158452310-827f9a99-86b1-46d6-b902-98bedd817e1d.png)
7. Press `Generate Registration Link` button and copy the url that appears. Do not open it yet.
8. Stop IAM.
9. Add this environment variables to the `iam` service:

```
KONG_ADMIN_GUI_SESSION_CONF: '{"secret":"secret","storage":"kong","cookie_secure":false}'
KONG_ENFORCE_RBAC: 'on'
KONG_ADMIN_GUI_AUTH: basic-auth
```

10. Start IAM again.
11. Open the link from (7).
 
![image](https://user-images.githubusercontent.com/5127457/158452981-f3f7d466-9067-4861-af33-d2493887a3ba.png)

12. Finish registration. 

![image](https://user-images.githubusercontent.com/5127457/158453035-2a524f22-9503-477c-98db-1cf07c4f7586.png)

13. Now your admin portal is allowing only authenticated access.
