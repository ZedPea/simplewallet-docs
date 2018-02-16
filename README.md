# Simplewallet
Simplewallet is most users portal to the TurtleCoin network. It is essentially a dumb pipe, which requests data from the RPC interface, TurtleCoind.exe, and then displays this data to the user when required.

Simplewallet unfortunately, along with a lot of the rest of the codebase, makes heavy use of global variables. They are fortunately only global to the .cpp/.h file, but it does make the code a lot harder to follow.

The two most important ones are `m_wallet`, and `m_node`. 

### m_wallet

`m_wallet` is the wallet itself, and is an implementation of the IWalletLegacy inteface, this implementation can be found in `src/WalletLegacy/WalletLegacy.cpp`
Some useful functions this class provide include:

* initAndGenerate - Makes a new wallet file with a password
* initWithKeys - Restores a wallet with a set of private keys and a password
* initAndLoad - Loads a wallet from a file with a stream to the file and a password
* getAccountKeys - Get the wallet public/private spend and view key
* save - Saves the wallet information back to the wallet file
* reset - Rewalks the blockchain looking for transactions belonging to the user - can optionally take a parameter to start walking from
* actualBalance and pendingBalance - pending balance is caused by transactions which haven't been confirmed yet, only included in a block. After a few subsequent blocks these will be moved into actual balance.
* sendTransaction

### m_node

`m_node` is your connection to the RPC interface, and lets you get more information than just the wallet can provide, such as block info, transaction info, and more. The relevant code can be found in `src/NodeRpcProxy/NodeRpcProxy.cpp`

Some useful functions this class provide include:

* getTransactions - Give it a vector of transaction hashes and it will give you all the information on them, you can then decode this with a users's private keys to determine which outputs belong to them
* getBlocks - The same as getTransactions, but with a block hash. Each block contains a vector of transactions.

Some of the functions in this class require a callback function - `const Callback& callback`. This is a bit intimidating if you don't know what it means. `Callback` is a typedef for `std::function<void(std::error_code)>;` - That's a function which returns void, and takes an error code as an argument.

When we call one of the functions which require a callback, upon the function completing, it will call this callback function if there was an error. So how the heck do we use it? Here's an example.

```
/* Here we set up the std::function and the magical async thread things.
   You can just copy these the next four lines into your program. */
std::promise<std::error_code> errorPromise;
std::future<std::error_code> f_error = errorPromise.get_future();
auto callback = [&errorPromise](std::error_code e) {errorPromise.set_value(e); };
m_node->addObserver(static_cast<INodeRpcProxyObserver*>(this));

/* Here we call the function we want to use. This function gets us some
   transaction details for each of the transaction hashes we give it. 
   We pass in the callback as an argument to the function. */
m_node->getTransactions(transactionHashes, transactions, callback);

/* Now, we check if the function returned an error */
auto error = f_error.get();

/* If there was an error, we can use the logger object to print it out nicely.
   More on this later. */
if (error)
{
    logger(ERROR, BRIGHT_RED) << "Failed to find transaction hash! Ensure you entered it correctly and your daemon is fully synced.";
    return false;
}
```

Whilst it is a bit finnicky, the only bit in this example you should need to change is the function you call, and of course what you do when an error occurs.

### Logger

So what's that logger object? It's just a nice way to print messages to the user with pretty colours and timestamps. You can see the colours and the different levels in `/src/Logging/ILogger.cpp`, some nice ones are GREEN, BRIGHT_YELLOW, BRIGHT_RED, AND BRIGHT_WHITE, but there's around 15 to pick from if you're feeling artistic...

The different logger levels allow log messages to be filtered depending upon how high the log level is. A message with FATAL priority will appear even when logging is turned down quite low. You can instrument your code with logger messages in TRACE or DEBUG mode to allow you to easily view what's going on, without spamming users's terminals with tons of messages.

It's very simple to use the logger object, we can just write to it the same way we would `std::cout`. For example:

```
} catch (const std::exception& e) {
  logger(ERROR, BRIGHT_RED) << "Failed to store wallet: " << e.what();
  return 1;
}
```

As you can see, we can send both string constants and string variables to it. We don't need to add a `std::endl` to the end, as each separate logger invocation is printed on a newline.

It'll look something like this when it actually gets printed out with all the formatting.

![logger example message](logger.png?raw=true "Logger example message")
