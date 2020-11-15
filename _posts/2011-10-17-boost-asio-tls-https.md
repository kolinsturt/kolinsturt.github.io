---
layout: post
category : lessons
tagline: "Boost Asio HTTPS on iOS"
title: Boost.Asio TLS experiments
description: My experience implementing the Boost Asio library for HTTPS on iOS
author: Kolin Stürt
tags : [C++, Boost.Asio, https, http, SSL, TLS]
redirect_from:
  - /lessons/2011/10/17/boost_asio_tls_https
---
{% include JB/setup %}

## TLS/SSL using Boost.Asio

Recently I needed to work on porting code that would perform HTTPS GET and POST network requests. I would port this to iOS. Most of the code was already in C++ but since the code did not need to stay portable, I took some advantage of [Grand Central Dispatch (GCD).](https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/index.html) 

 There are a lot of [samples](http://www.boost.org/doc/libs/1_47_0/doc/html/boost_asio/example/http/client/async_client.cpp) on how to use Boost.Asio to make regular HTTP requests.
 
Examples of TLS support are available for socket connections:

- [server.cpp](http://www.boost.org/doc/libs/1_42_0/doc/html/boost_asio/example/ssl/server.cpp)
- [client.cpp](http://www.boost.org/doc/libs/1_42_0/doc/html/boost_asio/example/ssl/client.cpp)

However there isn't ample information on how to convert regular HTTP request examples to TLS ones. Furthermore, there seems to be a lot of questions circulating on forums on how to accomplish this.

In order to do this task from the above [HTTP example](http://www.boost.org/doc/libs/1_47_0/doc/html/boost_asio/example/http/client/async_client.cpp), it requires converting the private member variable:

	tcp::socket socket_;

into:

	boost::asio::ssl::stream<boost::asio::ip::tcp::socket> socket_;

Therefor this blog is a walk through of my experience turning Boost’s HTTP request examples into an SSL example, while making it work on iOS. This code is by no means finished, but it's a test example.

Here is the [source code](https://github.com/CollinStuart/BoostAsioTLS) for the class with the converted `socket_` ivar.

For the header file:

	#ifndef test_NetworkManager_h
	#define test_NetworkManager_h
	
	#include <iostream>
	#include <istream>
	#include <ostream>
	#include <string>
	#include <functional>
	#include <asio.hpp>
	#include <bind.hpp>
	#include <boost/asio/ssl.hpp>
	
	using namespace std;
	using boost::asio::ip::tcp;
	
	enum NetworkManagerRequestType
	{
	    kNetworkManagerRequestTypeGet = 0,
	    kNetworkManagerRequestTypePOST = 1
	};
	
	class NetworkManager
	{
	public:
	
	    NetworkManager(boost::asio::io_service& io_service,
	                   boost::asio::ssl::context& context,
	                   const std::string& server,
	                   const std::string& path,
	                   std::function<void(string responseString)> callback,
	                   string postParams) : resolver_(io_service), socket_(io_service, context)
	    {
	        
	        socket_.set_verify_mode(boost::asio::ssl::verify_none); //use verify_peer for production code
	        
	        _callbackFunction = callback;
	        _isPost = false;
	        
	        // Form the request. We specify the "Connection: close" header so that the
	        // server will close the socket after transmitting the response. This will
	        // allow us to treat all data up until the EOF as the content.
	        std::ostream request_stream(&request_);
	        if (postParams.empty()) //GET request
	        {
	            request_stream << "GET " << path << " HTTP/1.0\r\n";
	            request_stream << "Host: " << server << "\r\n";
	            request_stream << "Accept: */*\r\n";
	            
	            request_stream << "Connection: close\r\n\r\n";
	        }
	        else //POST request
	        {
	            _isPost = true;
	            
	            request_stream << "POST " << path << " HTTP/1.0\r\n";
	            request_stream << "Host: " << server << "\r\n";
	            
	            request_stream << "Content-Type: application/x-www-form-urlencoded; charset=utf-8\r\n";
	            request_stream << "Content-Length: " + std::to_string(postParams.length()) + "\r\n\r\n";
	            request_stream << postParams << "\r\n\r\n";
	        }
	
	        
	        // Start an asynchronous resolve to translate the server and service names
	        // into a list of endpoints.
	        tcp::resolver::query query(server, "https");
	        resolver_.async_resolve(query,
	                                boost::bind(&NetworkManager::handle_resolve, this,
	                                            boost::asio::placeholders::error,
	                                            boost::asio::placeholders::iterator));
	    }
	    
	    //Use these to make network requests
	    static NetworkManager* getRequest(string urlString, string paramString, std::function<void(string responseString)> callback);
	    static NetworkManager* postRequest(string urlString, string pathString, string postParams, std::function<void(string responseString)> callback);
	    
	    
	private:
	    void handle_resolve(const boost::system::error_code& err,
	                        tcp::resolver::iterator endpoint_iterator);
	    
	    void handle_connect(const boost::system::error_code& err);
	    
	    void handle_handshake(const boost::system::error_code& error);
	    
	    void handle_write_request(const boost::system::error_code& err);
	    
	    void handle_read_status_line(const boost::system::error_code& err);
	    
	    void handle_read_headers(const boost::system::error_code& err);
	    
	    void handle_read_content(const boost::system::error_code& err);
	    
	    void _dispatchOnMainThreadWithResponseString(string dataString);
	    
	    tcp::resolver resolver_;
	    //tcp::socket socket_;
	    boost::asio::ssl::stream<boost::asio::ip::tcp::socket> socket_;
	    
	    boost::asio::streambuf request_;
	    boost::asio::streambuf response_;
	    
	    ostringstream _responseString;
	    std::function<void(string responseString)> _callbackFunction;
	    
	    bool _isPost;
	};
	
	#endif

Notice in the constructor you can pass a string of post parameters in the format *key1=value1&key2=value2*. HTTP 1.1 uses [chunked transfer-encoding](https://en.wikipedia.org/wiki/Chunked_transfer_encoding) which means that numbers will appear before and after the response. HTTP 1.0 does not do this, though usually requires a *Connection: close* in the header. For the post request case, I do not add a *Connection: close* header as you may not get the rest of the data if you do this. Also, the protocol depicts that after the last header before the post body there should be a */r/n/r/n* so the protocol knows the content is coming next. The last line of content should also have */r/n/r/n* so the protocol knows where the end of the content is.

There are two static class methods that you can use - `getRequest` and `postRequest`. Keeping somewhat in the style of modern network API's such as [AFNetworking](http://afnetworking.com/), these functions have callbacks where you can pass in a Lambda function, similar to callback blocks in Objective-C.

Inside the POST and GET requests, I am spawning a thread. Although Boost.Asio has an asynchronous model, I have found that it it's not tested and optimized for iOS. Instead I am using the standard thread library but you can use Grand Central Dispatch - GCD (more on this later for the callback functions).

	NetworkManager* NetworkManager::getRequest(string urlString, string paramString, std::function<void(string responseString)> callback)
	{
	    std::thread thread(_threadWork, urlString, paramString, "", callback);
	    thread.detach();
	    return NULL;
	}
	
	NetworkManager* NetworkManager::postRequest(string urlString, string pathString, string postParams, std::function<void(string responseString)> callback)
	{
	    std::thread thread(_threadWork, urlString, pathString, postParams, callback);
	    thread.detach();
	    return NULL;
	}

TODO: Add ability to get a pointer back of the request. Right now it's returning `NULL`. Boost calls the callback functions of the original Boost example - `handle_resolve`, `handle_connect`, `handle_handshake`, etc. Inside `handle_read_content`, I used GCD to perform the callback function back on the main thread:

	void NetworkManager::handle_read_content(const boost::system::error_code& err)
	{    
	    if (err == boost::asio::error::eof)
	    {
	        // Write all of the data that has been read so far.
	
	        _responseString << &response_;
	        
	        _dispatchOnMainThreadWithResponseString(_responseString.str());
	    }
	    else if (!err)
	    {
	        // Write all of the data that has been read so far.
	        //cout << &response_ << endl;
	        
	        //cout << _responseString.str() << endl;
	        _responseString << &response_;
	        
	        // Continue reading remaining data until EOF.
	        boost::asio::async_read(socket_, response_,
	                                boost::asio::transfer_at_least(1),
	                                boost::bind(&NetworkManager::handle_read_content, this,
	                                            boost::asio::placeholders::error));
	
	    }
	}


Take a look inside the `_dispatchOnMainThreadWithResponseString` and corresponding `Go` function:

	void Go(pair<function<void(string)>, string> *thePair)
	{
	    if (thePair)
	    {
	        string dataString = thePair->second;
	        function<void(string)> f = thePair->first;
	        f(dataString);
	        delete thePair;
	        thePair = NULL;
	    }
	}
	
	void NetworkManager::_dispatchOnMainThreadWithResponseString(string dataString)
	{
	    pair< function<void(string)>, string > *thePair = new pair< function<void(string)>, string >;
	    function<void(string)> f(_callbackFunction);
	    thePair->first = f;
	    thePair->second = dataString;
	    
	    //Pass in the callback function as context so it doesn't get destroyed
	    dispatch_async_f(dispatch_get_main_queue(), thePair, (dispatch_function_t)Go);
	}

If you use the [dispatch_async](https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/index.html#//apple_ref/c/func/dispatch_async) function here: As soon as `dispatch_async` executes, the `_callbackFunction` variable goes out of scope because the class deallocates after the request finishes. You need to hold on to the variable to still execute the callback function that the user passed in when making a network request.

One solution would be to copy the `std::function` into `dispatch_async_f` as a context. Although [std::function::target](http://en.cppreference.com/w/cpp/utility/functional/function/target) gives you a raw pointer to the function, the Lambda function is missing state such as capture variables so this method returns `NULL`. Therefore, package them together into a pair: The function wrapped as a `std::function` that includes scope, and the data string as the second part of the pair.
    
I used GCD because `std::thread` does not provide a way of dispatching a function on the GUI thread.

You can uncomment some of the `cout` statements for testing. If your coming from the Foundation (NSLog) world, note that `cout` to the standard console from multiple threads is not synchronized. Using `cout` with streams, especially without `std::endl` at the end can fill the buffer and `cout` may stop working. You can either flush or comment out the couts for streams.

        cout << std::flush;
        cout << _responseString.str() << endl;

Here are some examples of using the class interface:

	string paramString("device_id=12345");
    NetworkManager::postRequest("www.example.com", "/somepath/login.php", paramString, [this](std::string responseString)
        {
            cout << "The response is " << responseString << endl;
        });


	NetworkManager::getRequest("www.example.com", "/somepath/somecall.php", [this](std::string responseString)
	    {
	           cout << "Response is " << responseString << endl;
	     });

[Here](https://github.com/CollinBStuart/BoostAsioTLS) is the full code. This code is by no means complete. I would like to add a fail block and other error checking as well as certificate verification. (Replace `verify_none` with `verify_peer` for `set_verify_mode` and add your pem file to the project - need to call `set_verify_mode(boost::asio::ssl::verify_peer)` and `load_verify_file("ca.pem");` ) However I thought I would share it for anyone else who finds themselves in the same situation as I have. Since this blog is often dedicated to less common scenarios, it seems a perfect fit. For more general info see [Boost SSL](http://www.boost.org/doc/libs/1_47_0/doc/html/boost_asio/overview/ssl.html).
