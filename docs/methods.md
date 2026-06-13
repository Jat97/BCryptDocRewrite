# API Reference

BCryptJS offers multiple methods to help protect user credentials. The salt, hash, and compare methods can use callback functions as well.

To see how some of these methods work in practice, please view the tutorial.

## Callbacks

### Callback<T>: (err: Error | null, result?:T) => void

A basic function that returns an error on failure, and a value of T upon success.

### ProgressCallback: (percentage: number) => void

A function that returns the percentage of rounds completed in a hash. This percentage ranges from 0.0 to 1.0. This function can only be called for a maximum of once per 100 milliseconds, and can only be used with the compare and hash methods.

### RandomFallback: (length: number) => number[]

A function that is used to obtain random bytes when neither the Web Crypto API nor Node.js' built-in crypto library are available, such as when using an older browser. This callback is mostly unnecessary in everyday use.

## Methods

All functions that require a round as a parameter will default to 10 if the value is omitted.

### bcrypt.genSaltSync(rounds?: number): string

Synchronously generate a salt.

    bcrypt.genSaltSync(10);

### bcrypt.genSalt(rounds?: number): Promise<string>

Asynchronously generate a salt.

    await bcrypt.genSalt(10);

### bcrypt.truncates(password: string): boolean

Tests if a given password's length will exceed 72 bytes when hashed after being converted to UTF-8.

    bcrypt.truncates("password");

### bcrypt.hashSync(password: string, salt?: number): string

Synchronously generates a hash for the given password.

    bcrypt.hashSync("password", 10);

### bcrypt.hash(password: string, salt?: number): Promise<string>

Asynchronously generates a hash for the given password.

    await bcrypt.hash("password", 10);

### bcrypt.compareSync(password: string, hash: string): boolean

Synchronously compares a given password against a given hash.

    bcrypt.compareSync("password", hashWord); // true
    bcrypt.compareSync("not-password", hashWord); // false


### bcrypt.compare(password: string, hash: string): Promise<boolean>

Asynchronously compares a given password against a given hash.

    await bcrypt.compare("password", hashWord); // true
    await bcrypt.compare("not-password", hashWord); // false

### bcrypt.getRounds(hash: string): number

Returns the number of rounds used to encrypt the given hash.

    bcrypt.getRounds("password");

### bcrypt.getSalt(hash: string): string

Returns the salt portion of the given hash. This method does not validate the hash.

    bcrypt.getSalt(hashWord);

### bcrypt.setRandomFallback(random: RandomFallback): void

Sets the pseudo number generator to a random number as a fallback in the event that neither Web Crypto API or Node.js' built-in crypto library are available. 

If using this method, please ensure that the PRNG is secure and properly seeded to maintain cryptographic security.