# Using BCryptJS in a Project

BCryptJS can be paired with other libraries to further protect user credentials. This tutorial will use the Express-Validator and JSONWebToken libraries in conjunction with BCryptJS.

## Prerequisites

For the purposes of this exercise, it is assumed that you have a working ExpressJS backend and have downloaded both Express Validator and JSONWebToken. If neither library has been downloaded, you can use the following command in your terminal:

    npm install express-validator jsonwebtoken

## Salting and Hashing

In your account creation middleware, use Express Validator's `body` method to set the validation for your user data. To keep it simple, we're going to validate only usernames and passwords. Be sure to keep your validators at the top of your function!

    exports.create_account = [
        body('username', 'Please enter a username.').trim().custom(async user => {
            const account = await db.query(
                `SELECT * FROM users WHERE username = $1`,
                [user]
            );

            if(account.rows.length !== 0) {
                return await Promise.reject('This username is already in use.');
            }
        }),
        body('password', 'Please enter a password.').notEmpty().custom(async password => {
            if(password.length < 8) {
                return await Promise.reject('Your password is too short.');
            }
        }),
        body('confirm', 'Please re-enter your password.').notEmpty().isLength({min: 8}).custom(async (confirm, {req}) => {
            if(confirm !== req.body.password) {
                return await Promise.reject('The passwords do not match.');
            }
        }),

        (req, res) => {
            ...
        }
    ]

This code uses the PG library to send data to, and retrieve it from, a PostgreSQL database. If you're using a different database, simply replace any query calls in this tutorial with the relevant code from your database's library. You may also use different methods to help validate input data. Please consult Express Validator's documentation for the available methods.

After this, we can write the callback function for our middleware. If you want to retrieve any errors from the above validators, you need to include the `validationResult` method from Express Validator. We're going to include that, and to prevent BCrypt from running unnecessarily, we're going to put the rest of our code in a conditional.

    const errors = validationResult(req);

    if(!errors.isEmpty()) {
        res.status(400).json({errors: errors});
    }
    else {
        ...
    }
    
Now that we've included our validators and error handling, we can write the code to salt and hash a user's passwords, as well as create a JSONWebToken. While BCryptJS gives us the option to use the `getSalt` method, we're going to include the salt as a parameter for the hash method. To maintain optimal performance, the salt value will be 10. If you want to use a larger number, you may do so; however, be mindful that hashes will take longer to validate.

    bcrypt.hash(req.body.password, 10, async (err, hashWord) => {
        if(err) {
            return res.status(500).json({server_error: err});
        }
        else {
            const user = await db.query(
                `INSERT INTO users (username, password) VALUES ($1, $2)`,
                [req.body.username, hashWord]
            );

            jwt.sign(user, process.env.TOKEN_KEY, {expiresIn: new Date(Date.now() + 100000000)}, 
                (err, key) =>  {
                    if(err) {
                        return res.status(500).json({server_err: err});
                    }
                    else {
                        res.cookie('usercookie', key, {
                            expires: new Date(Date.now() + 100000000),
                            secure: false,
                            httpOnly: true,
                            path: '/api'
                        }).status(201).redirect('/api/home');
                    }
                });
        }
    });

You may configure your JWT however you wish, though it is recommended to change secure to true prior to deployment. It is important that you keep your token key in a .env file; for security reasons, it is best to include this file in .gitignore.

If you'd like, you can run your app and send input data to this endpoint. Once it's been submitted, check your database. If the password has been saved as a string of random characters, then it's successfully been hashed.

Now, we can move on to validating the hash.

## Validating the Hash

Since our login middleware will only be checking to see if the input data matches saved user data, we're going to use only BCryptJS and JWT. 

Before we can validate the hash, we need to write the code for error handling. Like in the account creation middleware, we're going to wrap everything in conditionals.

    exports.log_into_account = async (req, res) => {
        const user = await db.query(`SELECT * FROM users WHERE username = $1`, [req.body.username]);

        if(user.rows.length === 0) {
            return res.status(400).json({user_error: 'No account with this username exists.'});
        }
        else {
            if(req.body.password.length === 0) {
                return res.status(400).json({pass_error: 'Please enter a password.'});
            } 
            else {
                ...
            }
        }
    }

We can now write the code to validate user credentials using BCrypt's `compare` method. This method simply compares the given password with the hashed password for the given user. If there's a match, then we can use a callback function to provide authentication to the user.

    bcrypt.compare(req.body.password, user.rows[0].password, (err, result) => {
        if(err) {
            return res.status(500).json({server_err: err});
        }
        else if(result) {
            jwt.sign(user.rows[0], process.env.TOKEN_KEY, {expiresIn: new Date(Date.now() + 100000000)}, 
            (err, key) => {
                if(err) {
                    return res.status(500).json({error: err});
                }
                else {
                    res.cookie('usercookie', key, {
                        expires: new Date(Date.now() + 100000000),
                        secure: false,
                        httpOnly: true,
                        path: '/api'
                    }).sendStatus(200);
                }
            });
        }
        else {
            return res.status(400).json({pass_err: 'Your password is incorrect.'});
        }
    });

For a full list of BCryptJS' methods, please see the API Reference page.