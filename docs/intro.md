# BCryptJS

## What is BCrypt?

BCrypt is a password hashing function designed to protect the user's credentials. This is done by two processes. The first is salting, which assigns a unique string to the user's password, even if it's identical to another user's. The second is hashing, which converts the user's password into a random string of characters to obscure it from hackers.

## Installation

You can download BCryptJS via NPM:

    npm install bcryptjs

## How to Use

BCryptJS gives developers the option to hash passwords synchronously or asynchronously.

To hash a password asynchronously, you can simply add await:

    const salt = await bcrypt.genSalt(10);
    const hash = await bcrypt.hash("password", salt);

To hash a password synchronously, simply use the "sync" equivalents of the above methods:

    const salt = bcrypt.genSaltSync(10);
    const hash = bcrypt.hashSync("password", salt);

Alternatively, you can generate both the salt and the hash from BCrypt's hash and hashSync methods.

You can compare an input with the hashed password by using the compare method:

    await bcrypt.compare("password", hash);

## Security Considerations

One common problem in cybersecurity is what is called the "rainbow table attack," where attackers take precomputed tables and attempt to match hashes with passwords. By salting passwords before hashing them, BCrypt makes these types of attacks less effective.

BCrypt can be adjusted to increase the computational power required to create a hash. In practice, this causes the function to run at a slower pace, which in turn makes cyberattacks cumbersome. However, one potential drawback is that requests involving logging into, or creating, an account will take longer to process. 

The maximum length for inputs is 72 bytes, and the maximum for generated hashes is 60 characters. The input length is not implicitly checked by BCrypt, and must be checked using bcrypt.truncates(password).