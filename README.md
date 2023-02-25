# Password Protected file sharing systen

Application inspired by Web Dev Simplified to play with database schemas, node.js and express concepts

- To use it, you'll need node.js and mongoDB

1. `git clone` the repository or download the files
2. `cd` into the application folder
3. check the .env file to make sure you have your mongoDB URL correctly set and the desired port
4. open the terminal and type in `npm run devStart`
5. now you can open the application on your localhost:3000 and upload files to your mongo Database.

## Technologies

- Node.js
- MongoDB
- Express (routing and server-side logic)

initiate our package with all the defaults (-y flag)

```bash
npm init -y
```

## Dependencies

- Express
- Mongoose (to integrate with the Mongo Database)
- Multer (to handle file uploads because by default express doesn't handle it)
- ejs (as templating language for the views)
- bcrypt (to handle password hashing)
- dotenv (to use and hide environment variables)
- nodemon (as dev dependency "npm i --save-dev dependencyname" to handle hot reload of the server)

## Development Steps

- Add nodemon to our `package.json` as the start script

```json
"devStart": "nodemon server.js"
```

- Create the server file `server.js` and name our express instance as `app` so we can create routes like `app.get('/')` to our index page / localhost:3000 or `app.get('/file')` for example, also passing `req` (request) to handle our data gotten from the form and sent to the server and `res` (response) to handle what's sent back from the server to the user

In this case, on our index route all we have to send back is the rendered homepage

```js
app.get("/", (req, res) => {
  res.render("index");
});
```

- To tell our application where it will be started, we'll use our express instance we called `app` and call the `listen` method passing the port we want our application to run in

```js
app.listen(3000);
```

- Then we'll run our created script on the terminal to start the application

```bash
npm run devStart
```

- Now our server can be started but we need, first to tell it what our view engine is going to be

```js
app.set("view engine", "ejs");
```

- Then we need to create a view to be shown by it. First, inside our root directory we'll create a `views` folder and then an `index.ejs` file to contain our homepage

```html
<!-- views/index.ejs -->
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>pdr - File Sharing App</title>
  </head>
  <body>
    <form action="/upload" method="post" style="display: grid; gap: 0.5rem; grid-template-columns: auto 1fr; max-width: 500px; margin: 0 auto">
      <label for="file">File:</label>
      <input type="file" id="file" name="file" required />
      <label for="password">Password:</label>
      <input type="password" id="password" name="password" />
      <button style="grid-column: span 2" type="submit">Share</button>
    </form>
  </body>
</html>
```

- Now, with our homepage view created we'll setup the form's action, method and content type (enctype) so when the form is submitted it talks properly to our backend. We'll set the `method` as a "post" for post request, the `action` as '/upload' as the route that will receive the form's information when submitted and the `enctype` as 'multipart/form-data'

```html
<form
      method="post"
      action="/upload"
      enctype="multipart/form-data"
      ...
```

- Now it's time to setup the multer library so we can create our '/upload' route. On the `server.js` file, first we'll require the multer dependency, initialize it calling it "upload" and passing the `dest` (for destination) and tell it to send to our 'uploads' folder

```js
const multer = require("multer");

const upload = multer({ dest: "uploads" });
```

- The upload function we just created will serve as middleware, before we handle our request we'll upload a `single` file (that we gave the "name" property of "file" on our `index.ejs`)

```js
app.post("/upload", upload.single("file"), (req, res) => {
  res.send("");
});
```

### Connecting to our database using Mongoose

- First step is to require and use its `.connect()` method passing our database URL

```js
const mongoose = require("mongoose");
mongoose.connect(DATABASE_URL);
```

- We'll use environment variables to do so, that way we can use different URLs for different stages (development and production) as well as hide some personal data from our deployed version. To do so we'll require our `dotenv` dependency calling its `config()` method that tells our application to get our environment variables from `.env` files

```js
require("dotenv").config();
```

- Let's create a `.env` file on our root folder to hold our variables. We'll call them `DATABASE_URL` and `PORT` and pass it the values we want

```env
DATABASE_URL=mongodb://127.0.0.1/fileSharing
PORT=3000
```

- Reminder! To use the environment variables we'll use the "process.env.VARIABLE_NAME" syntax (so `process.env.DATABASE_URL` and `process.env.PORT` on our case)

- Now to our databse: let's create a folder called `models` and inside it a `File.js` that'll represent our File database object.
- First we'll require mongoose
- then create our schema
- then call mongoose's model() method passing the name of the "Table" and as the second parameter we'll pass our File schema
- and at the end export it so our File object can be used anywhere in our application

```js
const mongoose = require("mongoose");

const File = new mongoose.Schema({
  path: {
    type: String,
    required: true,
  },
  originalName: {
    type: String,
    required: true,
  },
  password: String,
  downloadCount: {
    type: Number,
    required: true,
    default: 0,
  },
});

module.exports = mongoose.model("File", File);
```

- Now to define our upload route, we'll create a `fileData` object with the structure we created on our schema and the contents of our form
- We can creade an if statement to make sure we only add a password if the user entered one that's not an empty string
- We can hash it using bcrypt passing the password and a number to let it auto-generate a salt and a hash to encrypt our password
- We'll use the `File` database instance to create an object passing the `fileData` object we created
- And we'll rerender our index view passing options like the file link so it can be copied and shared. To get the file link we'll get our domain origin from our `req`
- Remembering that as `bcrypt.hash()` and `File.create()` are async methods, so we'll change our function to be async so we can use the `await` keyword on it
- MongoDB has created the object and with it an objectId which we can use as a dynamic parameter on our file route so we can access the file through a URL composed by `/file` route and `/:id` and when that happens automatically start the file download

```js
app.post("/upload", upload.single("file"), async (req, res) => {
  const fileData = {
    path: req.file.path,
    originalName: req.file.originalname,
  };
  if (req.body.password != null && req.body.password !== "") {
    fileData.password = await bcrypt.hash(req.body.password, 10);
  }
  const file = await File.create(fileData);

  res.render("index", { fileLink: `${req.headers.origin}/file/${file.id}` });
});
```

- Now, as we passed the fileLink as option to our render, we can use it inside our `index.ejs` view and use the '<% %>' syntax to tell it to render on the server

```html
<% if (locals.fileLink != null) { %>
<div style="margin: 1rem auto">
  Your file has been uploaded:
  <a href="<%= locals.fileLink %>"><%= locals.fileLink %></a>
</div>
<% } %>
```

- So to get our file to be downloaded and our downloadCount to be changed we'll create another route on our `server.js` based on the dynamic paramater we passed in earlier (the id)
- we'll search our database for that file id getting the id from our `req.params.id` (since that's how we called it on our route), we'll increment its downloadCount, save it
- And prompt our user to download the file using the `download()` method from our `res` (response) passing the path to our file and the name we'll give it

```js
app.get("/file/:id", async (req, res) => {
  const file = await File.findById(req.params.id);
  file.downloadCount++;
  await file.save();

  res.download(file.path, file.originalName);
});
```

### Password protected files

- To handle password protected files we'll first tell express how to handle html forms with

```js
app.use(express.urlencoded({ extended: true }));
```

- Now, let's create a view to redirect the user to a 'password' view, so inside the view folder we'll create a `password.ejs` file

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Password | File Sharing App | pdr</title>
  </head>
  <body>
    <form method="post" style="display: grid; gap: 0.5rem; grid-template-columns: auto 1fr; max-width: 500px; margin: 0 auto">
      <label for="password">Password:</label>
      <input type="password" id="password" name="password" required />
      <button style="grid-column: span 2" type="submit">Download</button>
    </form>
  </body>
</html>
```

- And then we'll have to do simple if statements to check if our password is correct on our `server.js` file
- We'll check if the file has a password not null and also if the password from the uploaded file is also not null
- If there's a password, we'll redirect the user to the 'password' view
- Also, using bcrypt compare to check if the afterwards entered password is different than the one we registered before.

```js
if (file.password != null) {
  if (req.body.password == null) {
    res.render("password");
    return;
  }
  if (!(await bcrypt.compare(req.body.password, file.password))) {
    res.render("password", { error: true });
    true;
  }
}
```

- If it's different we'll pass down an error to use on our `password.ejs` same as we did with the file name on our `index.ejs` but now using the locals.error property we passed previously

```html
<% if (locals.error) { %>
<div style="color: red">Incorret Password</div>
<% } %>
```

- But to be able to download the file we'll have to make a post request, not a get, so we'll create a function to handle that protected file's download and put all of our `app.get('/file/:id', (req,res)=>{})` content inside it and change its use to simply

```js
app.route("/file/:id").get(handleDownload).post(handleDownload);

async function handleDownload() {
  const file = await File.findById(req.params.id);

  if (file.password != null) {
    if (req.body.password == null) {
      res.render("password");
      return;
    }
    if (!(await bcrypt.compare(req.body.password, file.password))) {
      res.render("password", { error: true });
      true;
    }
  }

  file.downloadCount++;
  await file.save();

  res.download(file.path, file.originalName);
}
```

- And finally our application should be good to go
