---
layout: post
title: "Using Google Data APIs with django apps"
description: ""
caregory:
tags: [Django, APIs, Google]
---
{% include JB/setup %}

####Setting up a Google Project
In order to use any of the Google APIs for your application, first you need to
set up a project in the [Google Developer's
Console](https://console.developers.google.com/project). Enable all the APIs
that you want to use in the APIs tab under APIs and auth. Under the Credentials
tab, create a Client ID and Client secret which is used for communication
between your application and the API. Enter all the allowed redirect urls in
the Redirect URIs field. These are the URLs to which the application redirects
after a user is successfully authenticated. You can change these URLs any time
you want. Now your Google App is ready for use.

We wanted to create a small [application](http://www.hackerearth.com/invite/) where users can invite their google
contacts to join [HackerEarth](http://www.hackerearth.com/).

####Dependencies
We use the [GData python client
library](https://pypi.python.org/pypi/gdata/2.0.18), which makes it easy to interact with
these Google services. You can install it using pip:
<br>
    
    sudo pip install gdata
or install it from
[source](https://code.google.com/p/gdata-python-client/downloads/list).

####Authentication
First we need to define some constants that we will use througout the
application
    
    #Obtained from Google Project Settings 
    GOOGLE_CLIENT_ID = <Your Client ID>
    GOOGLE_CLIENT_SECRET = <Your Client Secret>
    
    #Variable that specifies the data you want to access
    GOOGLE_SCOPE = "http(s)://www.google.com/m8/feeds/"

    #URL where the flow should go on successful authentication
    GOOGLE_APPLICATION_REDIRECT_URI = <Some URL>
    
    GOOGLE_REDIRECT_SESSION_VAR = <some arbitary value>

Since we wanted to use the Contacts API we used the above mentioned scope.
A list of other scopes is given
[here](https://developers.google.com/gdata/faq).
<br>
<br>
The first step for authentication is generation of an authentication token.
Since we have written a separate view to handle the auth token after successful
login, we are setting the authentication token in the session so that the same
token can be accessed in both the views.
    
    #Try to fetch the authentication token from the session
    auth_token = request.session.get('google_auth_token')
    
    #If an authentication token does not exist already,
    create one and store it in the session.
    if not auth_token:
        auth_token = gdata.gauth.OAuth2Token(
                client_id=GOOGLE_CLIENT_ID,
                client_secret=GOOGLE_CLIENT_SECRET, 
                scope=GOOGLE_SCOPE, 
                user_agent=USER_AGENT)
        request.session['google_auth_token'] = auth_token


####The login view
After successful authentication google returns a code which can be used to
generate an access token which acts as a confirmation of the authentication.
Here again we will set the authentication token on the session and redirect to
the same view where we came from.
    
    def google_login(request):
        
        #Fetch the auth_token that we set in our base view
        auth_token = request.session.get('google_auth_token')
        
        #The code that google sends in case of a successful authentication
        code = request.GET.get('code')
        
        if code and auth_token:
            #Set the redirect url on the token
            auth_token.redirect_uri = GOOGLE_APPLICATION_REDIRECT_URI
            
            #Generate the access token
            auth_token.get_access_token(code)
            
            request.session['google_auth_token'] = auth_token
            
            #Populate a session variable indicating successful authentication
            request.session[GOOGLE_COOKIE_CONSENT] = code
            
            #Redirect to your base page
            return redirect(request.session.get(GOOGLE_REDIRECT_SESSION_VAR))
        
        #If user has not authenticated the app   
        return redirect('wherever you want to')

####Making API calls
Now that our authentication token has an access token we can make the API calls to
fetch the data.

    if request.session.get(GOOGLE_C00KIE_CONSENT):
        
        #Create a data client, in this case for the Contacts API
        gd_client = gdata.contacts.client.ContactsClient()
        
        #Authorize it with your authentication token
        auth_token.authorize(gd_client)

        #Get the data feed
        feed = gd_client.GetContacts()

    else:
        
        #Since we want to get redirected back to the same page
        request.session[GOOGLE_REDIRECT_SESSION_VAR] = request.path
        
        #Generate the url on which authentication request will be sent
        authorize_url = auth_token.generate_authorize_url(
                redirect_uri=GOOGLE_APPLICATION_REDIRECT_URI)
        
        return redirect(authorize_url)

The flow will remain same for other APIs. The only things that will change are:
1. The scope URL
2. The GData client
    

P.S. I am the developer at [HackerEarth](http://www.hackerearth.com). 
Reach out to me at
virendra@hackerearth.com for any suggestion, bugs or even if you just want to
chat! Follow me [@virendra2334](https://twitter.com/virendra2334)

*Posted by Virendra Jain*
