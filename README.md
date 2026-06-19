# BCryptDocRewrite

A reimagining of the BCryptJS documentation for a technical writing portfolio. I am not affiliated with the BCryptJS library or its developers.

## BCryptJS

### What is BCrypt?

BCrypt is a password hashing function designed to protect the user's credentials. This is done by two processes. The first is salting, which assigns a unique string to the user's password, even if it's identical to another user's. The second is hashing, which converts the user's password into a random string of characters to obscure it from attackers.

### Installation

You can download BCryptJS via NPM:

    npm install bcryptjs

### How to Use

BCryptJS gives developers the option to hash passwords synchronously or asynchronously.

To hash a password asynchronously, you can simply add await:

    const salt = await bcrypt.genSalt(10);
    const hash = await bcrypt.hash("password", salt);

To hash a password synchronously, simply use the `Sync` equivalents of the above methods:

    const salt = bcrypt.genSaltSync(10);
    const hash = bcrypt.hashSync("password", salt);

Alternatively, you can generate both the salt and the hash from BCrypt's `hash` and `hashSync` methods.

You can compare an input with the hashed password by using the compare method:

    await bcrypt.compare("password", hash);

### Security Considerations

One common problem in cybersecurity is what is called the "rainbow table attack," where attackers take precomputed tables and attempt to match hashes with passwords. By salting passwords before hashing them, BCrypt makes these types of attacks less effective.

BCrypt can be adjusted to increase the computational power required to create a hash. In practice, this causes the function to run at a slower pace, which in turn makes cyberattacks cumbersome. However, this also has the drawback of increasing the time it takes to validate hashes.

The maximum length for inputs is 72 bytes, and the maximum for generated hashes is 60 characters. The input length is not implicitly checked by BCrypt, and must be checked using `bcrypt.truncates(password)`.

## API Reference

BCryptJS offers multiple methods to help protect user credentials. The `salt`, `hash`, and `compare` methods can use callback functions as well.

To see how some of these methods work in practice, please view the tutorial.

### Callbacks

#### Callback(T): (err: Error | null, result?:T) => void

A basic function that returns an error on failure, and a value of T upon success.

#### ProgressCallback: (percentage: number) => void

A function that returns the percentage of rounds completed in a hash. This percentage ranges from 0.0 to 1.0. This function can only be called for a maximum of once per 100 milliseconds, and can only be used with the compare and hash methods.

#### RandomFallback: (length: number) => number[]

A function that is used to obtain random bytes when neither the Web Crypto API nor Node.js' built-in crypto library are available, such as when using an older browser. This callback is mostly unnecessary in everyday use.

### Methods

All functions that require a round as a parameter will default to 10 if the value is omitted.

#### bcrypt.genSaltSync(rounds?: number): string

Synchronously generate a salt.

    bcrypt.genSaltSync(10);

#### bcrypt.genSalt(rounds?: number): Promise(string)

Asynchronously generate a salt.

    await bcrypt.genSalt(10);

#### bcrypt.truncates(password: string): boolean

Tests if a given password's length will exceed 72 bytes when hashed after being converted to UTF-8.

    bcrypt.truncates("password");

#### bcrypt.hashSync(password: string, salt?: number): string

Synchronously generates a hash for the given password.

    bcrypt.hashSync("password", 10);

#### bcrypt.hash(password: string, salt?: number): Promise(string)

Asynchronously generates a hash for the given password.

    await bcrypt.hash("password", 10);

#### bcrypt.compareSync(password: string, hash: string): boolean

Synchronously compares a given password against a given hash.

    bcrypt.compareSync("password", hashWord); // true
    bcrypt.compareSync("not-password", hashWord); // false


#### bcrypt.compare(password: string, hash: string): Promise(boolean)

Asynchronously compares a given password against a given hash.

    await bcrypt.compare("password", hashWord); // true
    await bcrypt.compare("not-password", hashWord); // false

#### bcrypt.getRounds(hash: string): number

Returns the number of rounds used to encrypt the given hash.

    bcrypt.getRounds("password");

#### bcrypt.getSalt(hash: string): string

Returns the salt portion of the given hash. This method does not validate the hash.

    bcrypt.getSalt(hashWord);

#### bcrypt.setRandomFallback(random: RandomFallback): void

Sets the pseudo number generator to a random number as a fallback in the event that neither Web Crypto API or Node.js' built-in crypto library are available. 

If using this method, please ensure that the PRNG is secure and properly seeded to maintain cryptographic security.

## Using BCryptJS in a Project

BCryptJS can be paired with other libraries to further protect user credentials. This tutorial will use the Express-Validator and JSONWebToken libraries in conjunction with BCryptJS.

### Prerequisites

For the purposes of this exercise, it is assumed that you have a working ExpressJS backend and have downloaded both Express Validator and JSONWebToken. If neither library has been downloaded, you can use the following command in your terminal:

    npm install express-validator jsonwebtoken

### Salting and Hashing

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

### Validating the Hash

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