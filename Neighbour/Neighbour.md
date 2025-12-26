## IDOR Basics
---

- In the description of this room, it is mentioned that we should look into `IDOR`, as this room contains similar content.
- Therefore, we can assume that an `IDOR` vulnerability is present in this room.
- `IDOR` stands for **Insecure Direct Object Reference**.
- This type of vulnerability occurs when a web application uses user-controlled input to directly access objects without proper authorization checks.

### Example:
---
- Imagine we just created a new bank account and view our profile.
- We notice the following `URL` in the browser:

```
http://online-service.thm/profile?user_id=1305
```

- We can see that a `user_id` parameter is being used.
- If we change this `user_id` to another number and are able to access data from other users’ bank accounts,
- then an `IDOR` vulnerability is present.

---

## Identifying the IDOR Vulnerability
---

- The room description also states that we should navigate to:

```
http://<Machine-IP-Address>
```

- So let’s do that:

![](Neighbour/images/Neighbour_1.png)

- We are presented with a login form and a message suggesting that we should check the source code if we do not have an account.
- Let’s inspect the source code.
- There, we find the following `HTML` comment:

```html
<!-- use guest:guest credentials until registration is fixed. "admin" user account is off limits!!!!! -->
```

- Based on this comment, we log in using the credentials `guest:guest`.

![](Neighbour/images/Neighbour_2.png)

- After logging in, we see a welcome message and a logout button.
- However, instead of interacting with the page, let’s take a closer look at the `URL`:

```
http://10.64.181.59/profile.php?user=guest
```

- We can see that `profile.php` uses a `user` parameter, which is currently set to `guest`.
- This looks like a potential `IDOR` vulnerability, as the user identity is controlled via the URL.
- Let’s revisit the comment we found earlier:

```html
<!-- use guest:guest credentials until registration is fixed. "admin" user account is off limits!!!!! -->
```

- An `admin` account is explicitly mentioned.
- So let’s try replacing the `user` parameter value from `guest` to `admin`.
- After doing this, we receive the following page content:

![](Neighbour/images/Neighbour_3.png)
