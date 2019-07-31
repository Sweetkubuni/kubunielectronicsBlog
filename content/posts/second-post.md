---
title: "Beast Boost WebSocket Server"
date: 2019-07-31T02:18:02-04:00
draft: false
---

Lately, I have been exploring the use of beast beast for a broadcast server.  Beast Boost is a HTTP and Websocket Library relying on boost asio and targets the C++11 standard. 

I used the following exampling code for my broadcast server : [websocket_server_sync.cpp].(https://github.com/boostorg/beast/blob/develop/example/websocket/server/sync/websocket_server_sync.cpp)

A safer method to passing the socket around is using shared_ptr. This prevents memory links or any runtime errors of losing scope of the data. 

    // This will receive the new connection
			auto socket_ptr = std::make_shared<boost::asio::ip::tcp::socket> ( ioc );

			// Block until we get a connection
			acceptor.accept(*socket_ptr);

			// Launch the session, transferring ownership of the socket
			std::thread { session_server::TelephoneSession(), socket_ptr, std::ref(router_queue) }.detach();

The biggest change I added to broadcast message to all connect clients was a routing thread that used the function object called Router : 

    	class Router {
	private:
		session_server_datastructure::threadsafe_queue<session_server_message::Message> & router_queue;
		std::map<std::string, session_server::TelephoneSession *> session_table;
		std::size_t id_counter = 0;

	public:
		Router(session_server_datastructure::threadsafe_queue<session_server_message::Message>& arg_router_queue);
		void operator () ();
		void Register(std::string username, TelephoneSession* arg_telephonesession);
		void Recieve(std::shared_ptr<session_server_message::Message>& message);
	};
This maps all connecting sessions to an incremented ID name, which I wanted the sessions to modify the label later on if I decide for private messaging.

The other class I made was called the telephone session class: 

    class TelephoneSession {
	private:
		std::string session_identity;
		boost::beast::websocket::stream<boost::asio::ip::tcp::socket> * ws_ptr;
	public:
		TelephoneSession();

		void operator () (std::shared_ptr<boost::asio::ip::tcp::socket> arg_socket, session_server_datastructure::threadsafe_queue<session_server_message::Message>& router_queue);
		void RecieveMessage(std::shared_ptr<session_server_message::Message>& message);
		void AssignIdentity(std::string arg_identidy);
	};
This handled receiving and sending of messages from the connect client.  The function object takes the tcp socket and moves it into the beast boost websocket wrapper. Within this wrapper, the handshake is handle for you. All you need to do is called the accept method.

A simple way to convert the incoming data from buffer to STL string is to use dynamic_buffer; an example would be the following : 

    std::string sbuffer;
	auto buffer = boost::asio::dynamic_buffer(sbuffer);
	ws.read(buffer);
	//do something with the data
	sbuffer.clear();

Once the router has mapped the  function object member to a label; it assigns the default Identity to the thread via AssignIdentity.

All Message contain a msg_type string property. The Message class requires any abstract class to contain a get method for any json data member it may have and also the convertjson ; this converts the class back into string format. I used the rapidjson library for its fast and reliable json parsing / encoding. 

If you would like to see the client and server code.  I provided the git projects below.

[websocket server](https://github.com/Sweetkubuni/websocket_server_cpp)

[websocket client](https://github.com/Sweetkubuni/websocket_client_1)
