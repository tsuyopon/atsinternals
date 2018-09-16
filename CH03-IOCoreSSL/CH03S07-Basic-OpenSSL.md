# Basics: OpenSSL


ATS's underlying support for TLS/SSL is implemented through the OpenSSL development library, so we need to first make a brief understanding of the OpenSSL development library.

The steps to use OpenSSL are as follows:
  - Create an SSL_CTX object with the SSL_CTX_new method
    - This object can be used to carry a variety of properties, such as:
      - Encryption and decryption algorithm
      - Session Cache
      - Callback
      - Certificate & private key
      - Other controls
  - After the network connection is created, the SSL object is created via SSL_new
    - This object can associate socket fd with it via the SSL_set_fd or SSL_set_bio methods.
    - This SSL object is required when you are working on an SSL connection later.
  - Complete the SSL handshake process using the SSL_accept or SSL_connect methods
  - Then use the SSL_read and SSL_write methods to receive and send data
  - Finally, use the SSL_shutdown method to turn off the TLS/SSL connection.

## BIO

OpenSSL proposes a concept of BIO, which corresponds to read/write and is therefore divided into Read BIO and Write BIO.

  - Read BIO and Write BIO can be set at the same time via SSL_set_bio()
  - Or set the Read BIO separately via SSL_set_rbio()
  - Or set Write BIO separately via SSL_set_wbio()

BIO can be associated with a memory block:

  - BIO_new (BIO_s_mem ())
  - BIO_new_mem_buf (ptr, len)

BIO can also be associated with a socket fd:

  - BIO_new_fd(int fd, int close_flag)
  - BIO_set_fd(BIO *b, int fd, int close_flag)

Or set socket fd directly on the SSL object:

  - SSL_set_fd
    - Set both read/write directions
  - SSL_set_rfd
    - Set only the socket fd to SSL object used to read the data
  - SSL_set_wfd
    - Set only the socket fd to SSL object used to write data

When setting socket fd to an SSL object, it is actually:

  - Created a BIO associated with the socket fd,
  - Then associate the BIO to the SSL object.
  - If the SSL pair is already associated with the BIO, it is first released via BIO_free().

In practice, BIO can be switched between memory blocks and socket fd at any time.

Other library functions of OpenSSL read and write resources through the BIO abstraction layer, for example:

  - SSL_accept and SSL_connect
  - SSL_read and SSL_write

We will find that BIO is very similar to VIO but different:

  - BIO abstracts access to memory blocks and socket fd, is a resource descriptor
  - VIO is connected to the socket fd and the memory block, and sets the direction of the data stream, which is a data pipeline.

## SSL handshake process

As the server side, you need to implement the SSL handshake process by calling SSL_accept. Before calling SSL_accept, you may need to make the following settings:

  - Associate certificate information to SSL objects
  - Set the SNI/Cert Callback function
  - Set the Session Ticket Callback function

As the client side, you need to implement the SSL handshake process by calling SSL_connect. Before calling SSL_connect, you may need to make the following settings:

  - Associate certificate information to SSL objects
  - Set up SNI
  - Set the NPN/ALPN parameters

## SSL data transfer process

After the SSL handshake process is completed, data transfer is possible:

  - SSL_read
    - Read information from Read BIO, decrypt it and write to the specified memory address
  - SSL_write
    - Get information from the specified memory address, write to Write BIO, send it after encryption

## SSL error message

After the OpenSSL API call, some error messages will be returned. The common error messages are explained as follows:

  - SSL_ERROR_WANT_READ
    - Indicates that EAGAIN was encountered while calling read()
    - Need to be called when the socket fd is readable
  - SSL_ERROR_WANT_WRITE
    - Indicates that EAGAIN was encountered while calling write()
    - Need to be called when the socket fd is writable
  - SSL_ERROR_WANT_ACCEPT
    - Indicates that a blocking condition was encountered while calling SSL_accept()
    - Need to be called when the socket fd is readable
  - SSL_ERROR_WANT_CONNECT
    - Indicates that a blocking condition was encountered while calling SSL_connect()
    - Need to be called when the socket fd is writable
  - SSL_ERROR_WANT_X509_LOOKUP
    - Indicates that SNI/CERT Callback requires the SSL_accept procedure to be suspended
    - And need to call SSL_accept() again to trigger the call to SNI/CERT Callback
    - Until the SNI/CERT Callback allows the SSL_accept process to continue
    - Re-invoke SSL_accept when appropriate, depending on business needs

The above error states are all encountered in the case of blocking, just need to recall the API that was called before at the appropriate time.

  - SSL_ERROR_SYSCALL
    - Indicates that a special error was encountered while executing the system call
    - The error type of the system call can be judged by the errno value.
  - SSL_ERROR_SSL
    - An error occurs in the OpenSSL library function, which usually indicates an error in the protocol layer.
  - SSL_ERROR_ZERO_RETURN
    - Indicates that the peer has closed the SSL connection.
    - But note that this does not mean that the TCP connection must be closed.

These are real errors and require corresponding error handling, such as closing the connection or retrying?

  - SSL_ERROR_NONE
    - Indicates that the operation is completed normally and there are no errors.

Only this is the call to complete a method without any errors.

## Create an SSL session in ATS

```
static SSL *
make_ssl_connection(SSL_CTX *ctx, SSLNetVConnection *netvc)
{
  SSL *ssl;

  if (likely(ssl = SSL_new(ctx))) {
    // Successfully created an SSL session via SSL_CTX
    // save a pointer to the SSL session in netvc
    netvc->ssl = ssl;

    // Only set up the bio stuff for the server side
    if (netvc->getSSLClientConnection()) {
      // If it is a connection between ATS and OS, set fd BIO directly on both sides of read and write.
      // because the probe protocol type is not required
      SSL_set_fd(ssl, netvc->get_socket());
    } else {
      // If it is a connection between Client and ATS, then it is more complicated:
      // First, you need to set a memory block BIO on the Read Side.
      //     This will allow you to read a portion of the data for probe
      // Then, set the fd BIO directly on the Write Side
      //     The OpenSSL API will only perform write operations if the read data conforms to the SSL protocol requirements.
      netvc->initialize_handshake_buffers();
      BIO *rbio = BIO_new(BIO_s_mem());
      BIO *wbio = BIO_new_fd(netvc->get_socket(), BIO_NOCLOSE);
      BIO_set_mem_eof_return(wbio, -1);
      SSL_set_bio(ssl, rbio, wbio);
    }

    // save the current netvc address to the SSL session
    //      Equivalent to the reverse pointer of the SSL session back to netvc
    // SSL_set_app_data is a macro equivalent to SSL_set_ex_data(ssl, 0 /* ssl_client_data_index */, netvc);
    // ssl_client_data_index is the return value of SSL_get_ex_new_index(),
    // and SSL_get_ex_new_index() is called by SSLInitClientContext()
    // and SSLInitClientContext() is called by SSLNetProcessor::start()
    // and SSL_get_ex_new_index() is only called once, so ssl_client_data_index is 0
    // So here for the SSL connection between ATS and OS and the SSL connection between Client and ATS, you can use:
    //      SSL_set_app_data(ssl, netvc) or SSL_set_ex_data(ssl, ssl_client_data_index, netvc) to set the association between netvc and SSL session
    //      SSL_get_app_data(ssl) or SSL_get_ex_data(ssl, ssl_client_data_index) to get the association between netvc and SSL session
    SSL_set_app_data(ssl, netvc);
  }

  // Return the created SSL session object
  return ssl;
}
```

I feel that ssl_netvc_data_index should be used instead of ssl_client_data_index. After all, both server and client are used.

# Reference material

- [OpenSSL Online Docs](https://www.openssl.org/docs/manmaster/ssl/SSL.html)
