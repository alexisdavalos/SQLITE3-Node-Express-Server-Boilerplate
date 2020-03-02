# Step By Step Guide: Node Express Server w/JWT User Authentication for SQLITE3 DB

1. create repository > clone it locally w/ `git clone [repoUrl]`
2. add .gitignore w/ `npx gitignore node || gitignore node`
3. add package.json w/ `npm init -y`
4. install boilerplate dependencies w/ `npm i [express, helmet, knex, dotenv, cors, sqlite3, jsonwebtoken, bcrypt]`
    ```js
    "dependencies": {
        "bcrypt": "^4.0.0",
        "cors": "^2.8.5",
        "dotenv": "^8.2.0",
        "express": "^4.17.1",
        "helmet": "^3.21.3",
        "jsonwebtoken": "^8.5.1",
        "knex": "^0.20.10",
        "sqlite3": "^4.1.1"
        },
    "devDependencies": {
        "nodemon": "^2.0.2",
    }
    ```
5. configure `scripts object` in `package.json` 
6. add "test" configuration to `package.json`
7. add "server" configuration to `package.json`
8. add "start" configuration to `package.json`   
 
    ```js
    "scripts": {
        "server": "nodemon index.js",
        "start": "node index.js"
    }
    ```
9. touch `index.js`: creates independent instance of `server.listen()`  
    ```js
    require("dotenv").config(); //access to .env variables
    const server = require('./server.js') 
    const PORT = process.env.PORT || 5000; 
    server.listen(PORT, ()=> console.log(`\n*** Running on port: ${PORT} ***\n`))
    ````
10. touch `server.js`: Declare server, routers, routes & sub-routes, sanity check endpoint
    ```js
    require("dotenv").config(); //access to .env variables
    const express = require('express');
    const helmet = require('helmet');
    const cors = require('cors');

    //routers
    const someRouter = require('./routerPath')

    //server instance
    const server = express();

    //middleware
    server.use(express.json());
    server.use(cors());
    server.use(helmet());

    //routes
    server.use('/api/someRouter', csomeRouter);
    
    //sanity check route 
    server.get('/', (req, res) =>{
        res.json({api: "up"});
    })

    //export module
    module.exports = server;
    ```
11. add `knexfile.js` w/ `knex init`
12. `knexfile.js` development environment configuration for `sqlite3`, this is the configuration that connects/creates the .db3 file or can link you to a postgres DB
    ```js
    const sqlite3 = {
        client: 'sqlite3',
            connection: { filename: './database/dev.db3' },
                useNullAsDefault: true,
            migrations: {
                directory: './database/migrations',
                tableName: 'dbmigrations',
            },
            seeds: { 
                directory: './database/seeds' 
            },
        };
    module.exports = {
        development: {
            ...sqlite3,
            connection: {filename:'./database/dev.db3'}, //creates dev database
        },
        testing: {
            ...sqlite3,
            connection: {filename:'./database/test.db3'} //creates testing database
        }
    }

    ```
13. mkdir `data` and add `dbConfig.js` or `connection.js` (up to you), this is the instance of knex that will initialize the settings in `knexfile.js`
    ```js
    const knex = require("knex");

    const knexfile = require("../knexfile.js");

    const env = process.env.NODE_ENV || "development";

    module.exports = knex(knexfile[env]); //sets configuration based on env variable

    ```
14. run `knex migrate:make create_Tables` to generate migrations file
15. configure the newly created file ex: `20200227172012_create_Tables.js`
    ```js
    exports.up = function (knex) {
        return knex.schema.createTable('users', users => {
            users.increments(); //gives auto icrementing ids
            users
                .string('username', 128) // username field
                .notNullable() //constraint
                .unique(); //constraint
            users
                .string('password', 128) //password field
                .notNullable(); //constraint
        });
    };

    exports.down = function (knex, Promise) {
        //drops the table (this mirrors the up function)
        return knex.schema.dropTableIfExists('users');
    };
    ```
16. run `knex migrate:latest` to rollout the new migration onto the db
    - there should be a brand new `database.db3` file in your `data` folder
    - `optional:` using SQLITE Studio and check for new table `users`
17. mkdir `users` and touch `users-model.js` and `users-router.js`
    - here we are creating some helper functions for use within our routers inside of `users-model.js`
    ```js
    //users-model.js
    const db = require('../data/dbConfig.js');
    //connects to the database via dbConfig.js

    function find() {
    return db('users').select('id', 'username', 'password');
    } //returns list of all users id's, usernames and password

    function findBy(filter) {
    return db('users').where(filter);
    } //return a list of users that match the filter

    async function add(user) {
    const [id] = await db('users').insert(user);
    return findById(id);
    }// adds a user and then returns the user added

    function findById(id) {
    return db('users')
        .where({ id })
        .first();
    }//find user by ID

    //exports user model functions
    module.exports = {
        add,
        find,
        findBy,
        findById,
    };
    ```
     - here we are creating our `users-router.js` which will have one endpoint to `find()` all users
    ```js
    //users-router.js
    const router = require("express").Router();

    const Users = require("./users-model.js");

    router.get("/", (req, res) => {
    Users.find()
        .then(users => {
        res.json(users);
        })
        .catch(err => res.send(err));
    });

    module.exports = router;
    ```
    - now update your `server.js` router and route declarations
    ```js
    //routers
    const usersRouter = require("../users/users-router.js");

    //declare route for usersRouter
    server.use("/api/users", usersRouter);
    ```
18. mkdir `auth` and touch `auth-router.js`
    ```js
    //auth-router.js
    const router = require("express").Router();
    const bcrypt = require("bcryptjs");
    const jwt = require("jsonwebtoken"); // <<< install this npm package

    const Users = require("../users/users-model.js");
    const { jwtSecret } = require("../config/secrets.js");

    // for endpoints beginning with /api/auth
    router.post("/register", (req, res) => {
    let user = req.body;
    const hash = bcrypt.hashSync(user.password, 10); // 2 ^ n
    user.password = hash;

    Users.add(user)
        .then(saved => {
        res.status(201).json(saved);
        })
        .catch(error => {
        res.status(500).json(error);
        });
    });

    router.post("/login", (req, res) => {
    let { username, password } = req.body;

    Users.findBy({ username })
        .first()
        .then(user => {
        if (user && bcrypt.compareSync(password, user.password)) {
            const token = generateToken(user); // get a token

            res.status(200).json({
            message: `Welcome ${user.username}!`,
            token, // send the token
            });
        } else {
            res.status(401).json({ message: "Invalid Credentials" });
        }
        })
        .catch(error => {
        console.log("ERROR: ", error);
        res.status(500).json({ error: "/login error" });
        });
    });

    function generateToken(user) {
    const payload = {
        subject: user.id,
        username: user.username,
        role: user.role || "user", //attaches 
    };

    const options = {
        expiresIn: "1h",
    };

    return jwt.sign(payload, jwtSecret, options);
    }

    module.exports = router;
    ```
19. touch `restricted-middleware.js` in `auth` directory where we will compare the token in the `authorization header` sent by the client to the one generated during <b>`POST` Login</b> 
    - first let's configure our `jwtSecret` by running `mkdir config` and then `touch config/secret.js`
    - this file will set our secret for our `token signature`
    ```js
    //secret.js
    module.exports = {
        jwtSecret: process.env.JWTKEY || "is it secret, is it safe?"
    };
    ```
    - read more about `jsonwebtoken` here: https://www.npmjs.com/package/jsonwebtoken
    - now for the `restricted-middleware.js` configuration:
    ```js
    //restricted-middleware.js
    const jwt = require("jsonwebtoken"); //npm module

    //require the secret.js file configured in previous step
    const { jwtSecret } = require("../config/secrets.js");

    module.exports = (req, res, next) => {
    const { authorization } = req.headers;

    if (authorization) {
        jwt.verify(authorization, jwtSecret, (err, decodedToken) => {
        if (err) {
            res.status(401).json({ message: "Invalid Credentials" });
        } else {
            req.decodedToken = decodedToken;
            //tokens match = pass to router
            next();
        }
        });
    } else {
        res.status(400).json({ message: "No credentials provided" });
        }
    };
    ```
20. touch `auth-router.js` in the `auth` directory, this router will hold our `POST` Register and `POST` Login `endpoints` as well as our `token generator`
    - we will be using `bcrypt` to hash registered users credentials
    - we will use `jsonwebtoken` to generate tokens whenever a user logs in
    ```js
    //auth-router.js
    const router = require("express").Router();
    const bcrypt = require("bcryptjs");
    const jwt = require("jsonwebtoken"); // <<< install this npm package

    const Users = require("../users/users-model.js");
    const { jwtSecret } = require("../config/secrets.js");

    // for endpoints beginning with /api/auth
    router.post("/register", (req, res) => {
    let user = req.body;
    const hash = bcrypt.hashSync(user.password, 10); // 2 ^ n
    user.password = hash;
    //once password is hashed with bcrypt, add user to database
    Users.add(user)
        .then(saved => {
        res.status(201).json(saved); //success
        })
        .catch(error => {
        res.status(500).json(error); //failure
        });
    });

    router.post("/login", (req, res) => {
    let { username, password } = req.body;

    Users.findBy({ username })
        .first()
        .then(user => {
        //compares credentails sent in header to db records
        if (user && bcrypt.compareSync(password, user.password)) {
            const token = generateToken(user); // get a token

            res.status(200).json({
            message: `Welcome ${user.username}!`,
            token, // send the token
            });
        } else {
            res.status(401).json({ message: "Invalid Credentials" });
        }
        })
        .catch(error => {
        console.log("ERROR: ", error);
        res.status(500).json({ error: "/login error" });
        });
    });

    function generateToken(user) {
    const payload = { //payload sent to client
        subject: user.id,
        username: user.username,
        role: user.role || "user" 
        //adds role property to user in token payload
    };

    const options = {
        expiresIn: "1h", //sets expiration time for token
    };

    return jwt.sign(payload, jwtSecret, options); //returns token
    }

    module.exports = router;

    ```
21. add `checkRole()` to `server.js`
    - checks the role of the user trying to access the resource
    ```js
    //update usersRoute
    server.use("/api/users", restricted, checkRole("user"), usersRouter);

    //add to bottom of server.js
    function checkRole(role) {
        return (req, res, next) => {
            if (
            req.decodedToken &&
            req.decodedToken.role &&
            req.decodedToken.role.toLowerCase() === role
            ) {
            next();
            } else {
            res.status(403).json({ you: "shall not pass!" });
            }
        };
    }
    ```
21. add new `auth-router.js`and `retriscted-middleware.js` to `server.js`
    ```js
    //routers
    const usersRouter = require('../users/users-router.js');
    const authRouter = require('../auth/auth-router.js');
    const restricted = require('../auth/restricted-middleware.js');

    //routes
    server.use('/api/auth', authRouter)
    server.use('/api/users', restricted, usersRouter)
    ```
22. At this point we're ready to check our endpoints and create some users!
    - On Postman or Insomnia make the following requests:
        - <b>POST `/api/auth/register`</b>
            ```js
            {
                "username": "user",
                "password": "password"
            }
            ```
        - <b>Server should respond with:</b>
            ```js
                {
                    "id": 1,
                    "username": "user",
                    "password": "$2a$10$l2jRhdwkrHddCUxB9j0KM.qrdVEUpQAbzKTH6zkAyFHUEfGGnJyJS"
                }   
            ```
        - <b>Now POST `/api/auth/login` </b>
            - <b>Server should respond with: </b>
            ```js
                {
                    "message": "Welcome user!",
                    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWJqZWN0IjoxLCJ1c2VybmFtZSI6InVzZXIiLCJyb2xlIjoidXNlciIsImlhdCI6MTU4Mjg0ODAwNiwiZXhwIjoxNTgyODUxNjA2fQ.jpqtue3zxSyotOO-Wg0CL_h5B6K3buT2lQ1bnotxye8" //token
                }
            ```
23. Once we've received a token from the server, we have to check our `restricted usersRouter.js` in `server.js`
        ```js
        server.use('/api/users', restricted, usersRouter)
        ```
    - On Postman or Insomnia make the following requests:
        - <b>GET `/api/users`</b> 
        - <b>Server should respond with:</b>
        ```js
        {
            "message": "No credentials provided"
        }
        ```
        - <b>This is because we need to pass our `token` into the `headers` as an `authorization` key value pair ex: </b>   
        ```js
        {
            "authorization": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWJqZWN0IjoxLCJ1c2VybmFtZSI6InVzZXIiLCJyb2xlIjoidXNlciIsImlhdCI6MTU4Mjg0ODAwNiwiZXhwIjoxNTgyODUxNjA2fQ.jpqtue3zxSyotOO-Wg0CL_h5B6K3buT2lQ1bnotxye8"
        }
        ```
        - <b>Now we can try to GET `/api/users` and we should see an array of users ex: </b>
        ```js
        [
            {
                "id": 1,
                "username": "user",
                "password": "$2a$10$8CnZpkejFvQ5Jx7enZQuEeJ2.vR9QZLZ5AoSbQQ4m4CxV7WYoruMm"
            },
            {
                "id": 2,
                "username": "user2",
                "password": "$2a$10$eZ6n2uarYf68KGKGBfBx0u0Ie5MOo3UTwAiIVh64bR2MFRGldgi0O"
            },
            {
                "id": 3,
                "username": "user3",
                "password": "$2a$10$G6rg8DyHlxyRAjiE64UfS.re9CObDpyrAW.Nx0DbPBY6X/GioGspS"
            }
        ]

        ```     
## Features

- Creates a user using the information sent inside the body of the request. Hash the password before saving the user to the database.
- Use the credentials sent inside the body to authenticate the user. On successful login, create a new JWT with the user id as the subject and send it back to the client. If login fails, respond with the correct status code and the message: 'You shall not pass!'
- If the user is logged in, respond with an array of all the users contained in the database. If the user is not logged in respond with the correct status code and the message: 'You shall not pass!'.

## ENDPOINTS

| Feature                | Method | URL                         |
| :--------------------- | :----- | :------------------------   |
| Creates a user         | POST   | /api/auth/register          |
| Authenticates user     | POST   | /api/auth/login             |
| List of users in DB    | GET    | /api/users                  |
