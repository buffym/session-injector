Overview
========

If you are developing an application that serves subdomains, the `:all` cookie store domain parameter will most likely serve your needs.  However if your application serves distinct domains, you will most likely encounter some difficulties, as secure browsers will not accept "third party cookies" (e.g. any cookies you issue for a different domain will be disregarded).

There are a couple of approaches, neither of which are particularly elegant: http://stackoverflow.com/questions/263010/whats-your-favorite-cross-domain-cookie-sharing-approach

This gem provides a middleware that implements a "handshake" protocol based on a token inserted into a URL parameter, which allows you to transparently re-establish a Rack/Rails session accross domains.

Usage
=====

If you are using Rails, insert this into your `config/application.rb`:

    config.middleware.insert_before ActionDispatch::Cookies, "Rack::Middleware::SessionInjector", :key => '_your_session'

There are three public methods through which you can initiate the session transfer:

    Rack::Middleware::SessionInjector.generate_handshake_token(request, target_domain, lifetime = nil)
    Rack::Middleware::SessionInjector.generate_handshake_parameter(request, target_domain, lifetime = nil)
    Rack::Middleware::SessionInjector.propagate_session(request, target_domain, lifetime = nil)

you can append the parameter to a link:

    link_to "http://otherdomain?#{Rack::Middleware::SessionInjector.generate_handshake_parameter(request, 'myotherhost')}"

or tell the middleware to rewrite the Location header on an HTTP redirect response:
 
    Rack::Middleware::SessionInjector.propagate_session(request, 'myotherhost')

or you can just generate the token and use some custom method to convey it the request on the target domain:

    token = Rack::Middleware::SessionInjector.generate_handshake_token(request, 'myotherhost')

Security
========

The "handshake" token is generated via `ActiveSupport::MessageEncryptor` using a dynamically generated key (although you can specify a static key yourself).

The token data consists of:

    handshake = {
      :request_ip => request.ip,
      :request_path => request.fullpath, # more for accounting/stats than anything else
      :src_domain => request.host,
      :tgt_domain => target_domain,
      :token_create_time => Time.now.to_i,
      # the most important thing
      :session_id => extract_session_id(request, session_injector.session_id_key)
    }

This token is verified in the following manner:

* client request ip must match
* target domain must match
* token must not be older than receiver-specified lifetime
* token must not be older than sender-specified lifetime
