# Backend Testing
Backend Testing will be considered a form of Integration Testing we are testing the contract between our route handlers, queries and database. And making sure they work well together in turn assuring us that our backend works as we expect it.

Today we will be testing a Backend Application that handles users and notes. Some feature of this backend include:

* User Authentication:
  * A user can signup with username and password
  * A user can login
  * A user can logout
* Notes
  * Unauthenticated users can post anonymous notes.
  * Any unauthenticated user can retrieve all public notes.
  * Authenticated users can add a note.
  * Authenticated users can get the notes they have added.
  * Anonymous notes are public by default.

[This is the Backend](https://github.com/joinpursuit/Pursuit-Core-Web-Backend-Testing-Starter) we will use, take a look at its README as well its files, make sure there is nothing surprising or confusing about it. Try a few requests with Postman.

## Setup

1. Clone starter Backend App and install dependencies:
  ```
  git clone https://github.com/joinpursuit/Pursuit-Core-Web-Backend-Testing-Starter.git
  cd Pursuit-Core-Web-Backend-Testing-Starter
  npm install
  ```

2. Install dev dependencies

  ```
  npm install -D jest jest-extended @types/jest supertest 
  ```

  * [`jest`](https://jestjs.io/) - Our testing framework
  * [`jest-extended`](https://github.com/jest-community/jest-extended) - Provides additional handy matchers
  * [`@types/jest`](https://www.npmjs.com/package/@types/jest) - Jest Typescript Typings. This is nice to give use intellisense/autocomplete for jest
  * [`supertest`](https://github.com/visionmedia/supertest) - HTTP client for making HTTP assertions. You can think of this as the `axios` for `testing`. 

3. Save `jest.config.js`

  ```js
  module.exports = {
    setupFilesAfterEnv: ["jest-extended"],
    testEnvironment: "node"
  }
  ```

  This ensures that we get the additional matchers from `jest-extended`. Also sets the jest test environment config to be `node` (default is `jsdom`)

4. Set/add `test` script to `package.json` as shown here:

  ```
  "test": "jest --runInBand --watch"
  ```

 This makes sure that our tests run serially (`--runInBand`). Normally for unit testing Jest will run tests in parallel for speed and efficiently but since our tests will share the same database if one test modifies a record another might utilize the results of the previous test or conflict with it giving us have hard to debug issues. `--watch` will keep jest running so that we get hot test reload when we change a source file or test file.

5. Create a database and seed it for the first time
```
createdb backend_testing_users_db
psql -f db/seed.sql -d backend_testing_users_db 
```

6. Create a `resetDb.js` file with the following content. Save it to the `db/` directory
  ```js
  const { execSync } = require('child_process')

  const SEED_FILE_PATH = __dirname + "/seed.sql"
  const DATABASE_NAME = "backend_testing_users_db"

  const resetDb = () => {
    execSync(`psql -f ${SEED_FILE_PATH} -d ${DATABASE_NAME}`)
  }

  module.exports = resetDb;
  ```

  This file calls `psql` which will execute our seed file `-f ${SEED_FILE_PATH}` in the context of our database `-d ${DATABASE_NAME}`. Note that our seed file should not include the lines to `DROP`/`CREATE` a database or connect to it.

  We will use this file to spin up and down our database tables in between tests. We want to avoid our tests using shared state (global variables, database data, etc) as much as possible.

## Hands On
### Explore the Backend to be Tested
Once you have cloned the starter App take 15 minutes to read the Backend Docs and familiarize yourself with the endpoints and requests.

### Our First Test. `/api/notes/public` retrieves all notes
Let us test endpoints or routes that do not require a user to be logged in first. We will test that a client can retrieve all public  notes when a `GET` request to `/api/notes/public` is made.

1. Create a `__tests__` folder at the root of your project.
2. Create a `notes.test.js` test file
3. Add the following content
```js
const request = require('supertest')
const app = require('../app')
const reqAgent = request.agent(app)
const resetDb = require('../db/resetDb');

beforeEach(() => {
  resetDb()
})

afterAll(() => {
  resetDb()
})

describe('/api/notes endpoints', () => {
  test('All public notes are retrieved', async () => {
    const { status, body } = await reqAgent.get('/api/notes/public')
    expect(status).toBe(200)
    expect(body).toContainAllKeys(['err', 'msg', 'payload'])
    expect(body.err).toBeFalse()
    expect(body.msg).toMatch(/retrieved all public notes/i)

    const notes = body.payload
    expect(notes).toBeArrayOfSize(2) // There are only 2 public notes in our database

    // Sample and test values of one note
    expect(notes[0]).toContainAllKeys(['id', 'created_at', 'user_id', 'text', 'is_public'])
    expect(new Date(notes[0].created_at)).toBeValidDate()
    expect(notes[0].id).toBeNumber()
    expect(notes[0].user_id).toBeNumber()
    expect(notes[0].text).toBeString()
    expect(notes[0].is_public).toBe(true)
  })
})
```

#### Explanation
* `beforeEach` will execute the callback provided after each test. In this case the callback invokes `resetDb()` to reset our database after each test. Remember that we don't want to share state (data, variables etc) between tests as possibnle. Read the `beforEach` docs [here](https://jestjs.io/docs/en/api#beforeeachfn-timeout)
* `afterAll`. After all the tests make sure our database is reset and does not contain any test data.
* `reqAgent.get('/api/notes/public')`. Is what will fire the request to our our app's endpoint. Kind of like `axios` but for testing. You can see that `reqAgent` is made by importing `request` from `supertest` and creating an agent by passing in our app.
* `const { status, body }` is destructuring the status and body of the response so that we can make assertions on them.
* `expect(status).toBe(200)`. It reads pretty easily doesn't it?. We are expecting that for this request the response `status` was a successful `200`
* `expect(body).toContainAllKeys(['err', 'msg', 'payload'])`. In this way you can check that your response object is returning all the keys that you expect.
* Then you can make assertions on the values of the response body properties like so:
```js
  expect(body.err).toBeFalse()
  expect(body.msg).toMatch(/retrieved all public notes/i)
```
* The rest is us sampling and checking the response `payload` property. For easier readability we save `body.payload` in a variable `notes` (`const notes = body.payload`). For this request the payload will contain an array of note objects. In this case we should get 2 notes because that's the number of notes we start with in our `seed.sql` file. We also take a look at the first note and assert that it has properties `id`, `created_at`, `user_id`, `text` and `is_public`. Lastly we assert that those properties have the expected values like, expecting that the note `text` property is a string (`expect(notes[0].text).toBeString()`)

### Your turn. `/api/users` retrieves all users
1. Create a `users.test.js` file inside `__tests__/`
2. Write a test that verifies that when a request to `/api/users` is made all the users are returned.
    * **DO NOT COPY AND PASTE CODE**. Build muscle memory.
    * Make sure you have the correct `require`s and setup.
    * This will be a similar and even simpler test than the previous one.
    * Implement your test:
  ```js
  describe('/users endpoints', () => {
    test('GET to /api/users retrieves all users', async () => {
      // Your testing code goes here
    })
  })
  ```
<details>
  <summary>Compare your approach to this solution. </summary>

How does yours differ from this? Ask any questions came up?
```js
// users.test.js 
const request = require('supertest')
const app = require('../app')
const reqAgent = request.agent(app)
const resetDb = require('../db/resetDb');

afterEach(() => {
  resetDb()
})

afterAll(() => {
  resetDb()
})

describe('/users endpoints', () => {
  test('GET to /api/users retrieves all users', async () => {
    const { status, body } = await reqAgent.get('/api/users')
    expect(status).toBe(200)
    expect(body).toContainAllKeys(['err', 'msg', 'payload'])

    const users = body.payload
    expect(users).toBeArrayOfSize(2) // Our seed file inserts two users. So we have 2 users in our database.

    // Sample one user and assert property values
    const user = users[0]
    expect(user).toContainAllKeys(['id', 'username'])
    expect(user.id).toBeNumber()
    expect(user.username).toBe('JonSnow') // JonSnow is the first user in our database seed file
  })
})
```
</details>

### Test that a new anonymous note can be added by posting to `/api/notes/anonymous` 
Ok, we tested retrieving data from our API. How do we test adding/posting data?
Take a look at the following code:
```js
test('A new anonymous note can be posted', async () => {

  let newNote = {
    text: "This is an anon note. Not associated with any user",
  }

  const { status, body } = await reqAgent.post('/api/notes/anonymous').send(newNote)
  expect(status).toBe(200)
  expect(body).toContainAllKeys(['err', 'msg', 'payload'])
  expect(body.err).toBeFalse()
  expect(body.msg).toMatch(/added new anonymous note/i)

  const note = body.payload
  expect(note).toContainAllKeys(['id', 'created_at', 'user_id', 'text', 'is_public'])
  expect(note.id).toBeNumber()
  expect(note.user_id).toBe(null)
  expect(note.text).toBe(newNote.text)
  expect(note.is_public).toBe(true)
  expect(new Date(note.created_at)).toBeValidDate()
})
```
#### Explanation
* To post/send data we use `reqAgent.post(url).send(data)`. Compare to example when getting data instead.
* It's important to never forget the `await` for asynchronous code. If we do our assertions will fail
* Assert that our response has the expected keys and tha the values are what we expect, for instance we expect the `user_id` to be null because this is an anonymous note. We also expect the response note `text` property to be the same that we sent (`expect(note.text).toBe(newNote.text)`)

### Your turn. Test that a new user can be signed up by posting to `/api/auth/signup`
* We could do this in `users.test.js` but I recommend having a new file `auth.test.js` instead. `auth.test.js` will be for user authentication tests only.
* Create the `__tests__/auth.test.js` test file
* Make sure you have the correct `require`s and setup.
* This test involves sending/posting data just like our previous test.
* Assert that your response was successful by checking the status code and the payload similarly to our previous test. If in doubt try this request in Postman first.
* Implement your test
  ```js
  describe('/api/auth endpoints', () => {
    test('POST to /signup registers a user and sends user back along with success message', async () => {
      // Your testing code goes here
    })
  })
  ```
<details>
  <summary>Compare your approach to this solution. </summary>

How does yours differ from this? Ask any questions came up?
```js
// auth.test.js
const app = require('../app')
const request = require('supertest')
const resetDb = require('../db/resetDb')
const reqAgent = request.agent(app)

beforeEach(() => {
  resetDb()
})

afterAll(() => {
  resetDb()
})

describe('/api/auth endpoints', () => {
  test('POST to /signup registers a user and sends user back along with success message', async () => {
    const user = {
      username: 'TestUser837',
      password: "s3cr37"
    }

    const { status, body } = await reqAgent.post('/api/auth/signup').send(user)
    expect(status).toBe(200)
    expect(body).toContainAllKeys(['err', 'msg', 'payload'])
    expect(body.err).toBeFalse()
    expect(body.msg).toMatch("User successfully signed up")
    expect(body.payload.username).toBe(user.username)
    expect(body.payload.id).toBeNumber()
    expect(body.payload.password).toBeUndefined() // Make sure password was not returned
  })
})
```
</details>


## Additional Resources
* [Jest - An Async Example](https://jestjs.io/docs/en/tutorial-async#asyncawait)
* [Jest Cheatsheet](https://devhints.io/jest)
