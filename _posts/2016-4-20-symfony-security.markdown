---
layout: post
title:  "Understanding the Symfony Security Component"
date:   2016-4-20 00:25:00
categories: Symfony
---

<br>

## What's my purpose for writing this article?

So I am going to guess that the majority of Symfony 2 developers are similar to myself in that they make extensive use of the security component that ships with the framework, but don't really know too much about how it really works. My experience with the security component boils down to fiddling with the configuration now and then, writing some Voters, and using the AuthorizationChecker service. So every once in a while, when I found myself having to make some changes to my security.yml I realized that if ever I got an error or unexpected behavior I felt like I had very little understanding of why it happened, so I'd just fiddle with the configuration settings until I got it back to doing what I wanted. 

The whole process felt very blind, even after reading through the majority of the documentation on the component multiple times. I didn't have a lot of confidence in what I was commiting. Sure I manually tested it to verify it was working and also had my integration tests, but I still wasn't satisfied. So my intention when writing this article was to better understand exactly what the security component was doing behind the scenes, both so I could understand how it turned my configuration into a working security layer and so that I have a sounder mental model when working with it.

## Who's this article targeted towards?

This article was written with the intenion of being understandable by somebody with no prior Symfony 2 experience. That that said, it shouldn't be a shock that the purpose of this article is not to teach you how to use the Symfony security component, at least not directly. For that, the Symfony book already does a fantastic job and if thats what you are looking for I recommend you check it out [here](http://symfony.com/doc/current/book/security.html). Instead, this article is meant to provide a clear understanding of both how and why the security component does what it does by explaining exactly what is happening inside of it from start to finish, going through what each of the different pieces do, and how all those pieces tie together. 

Last disclaimer I'd like to add is that while I've been using the fullstack framework for a few years now I am definitely not an expert on the internals of Symfony components. All of the following was written as I manually traversed my way through the security component, supplemented by the amazing documentation in the Symfony book and cookbook. If you notice anywhere I am mistaken or unclear please let me know.

## Authentication:

### From the top:

So for those unfamiliar with Symfony's security component, the [documentation on the security component itself](http://symfony.com/doc/current/components/security/firewall.html#flow-firewall-authentication-authorization) does a great job of explaining the flow of the security process:

1. Firewall is triggered, initializing security process.
2. Authentication listeners try to perform authentication using the current Request object.
3. Assuming a listener successfully authenticated a *token* (not a user, sometimes an import distinction), the authentication process is now over.
4. You now have the ability to use *Authorization* to restrict access to certain resources.

Below I will go into each one of these steps in detail, describing exactly what happens under the hood and why.

Since the firewall is the starting point for the authentication process, it makes more sense to define exactly what it means to be authenticated before we go into what the firewall itself does.

### What does it mean to be authenticated?

Being authenticated simply means that the Symfony application is, at least to some degree, aware of who is currently using it. (A specific user? An anonymous user? An api client?).

* Authentication always begins with triggering the firewall. Unless disabled, the Firewall will be triggered on every request since it is nothing more than an event listener. Triggering the firewall may or may not actually result in authentication being performed, depending on if the current request is one which requires authentication. Whether or not a request require's authentication is determined by the authentication listeners, which will be covered below.
* The only difference between a request that requires authentication and one that doesn't is that a request which require's authentication to occur is one that requires knowledge of WHO/WHAT is making the request.
* A request that require's authentication doesn't necessarily imply that its attempting to access something which is restricted, although thats one situation when authentication would be necessary. When a request require's authentication it simply means that, for whatever reason, the application requires knowledge about who is making the request.
* Many times this knowledge is required because access to certain actions or data is restricted and we need to know who they are to determine if they should be allowed or not.
* Alternatively, maybe the application just wants to keep of log of who did what and so needs to know who the current user is.
* Regardless of why its required, authentication is always triggered the same way, by the firewall. 

### What is the firewall?

The firewall is simply the tripwire that initiates the Authentication process.

* During every request a call to `Symfony\Component\Security\Http\Firewall::onKernelRequest`. As you may have guessed, this method is registered in the Symfony event listener for the 'kernel.request' event.
* It is important to remember that just because a call is made to `Symfony\Component\Security\Http\Firewall::onKernelRequest` is made doesn't mean that the authentication process will actually occur. It just means that the firewall will check IF authentication should occur or not. If authentication is supposed to occur the firewall will then proceed with the authentication process.
* This is done by listeners registered with the firewall. There are various reasons why something may want to listen on the firewall other than initiating authentication. For example: logging out a user is not authenticating them but still requires listening on the firewall. For now though, for simplicity's sake we will only concern ourselves with the Authentication Listeners (authentication listeners are just firewall listeners that may initiate authentication).
    * Firewall Listener Interface: `Symfony\Component\Security\Http\Firewall\ListenerInterface`
    * Authentication Listener Base Class: `Symfony\Component\Security\Http\Firewall\AbstractAuthenticationListener`
* Authentication occurs when one of the AuthenticationListeners on the firewall actually decides to perform authentication. If none of the authentication listeners attempt to actually perform authentication, then no authentication occurs because none was required for this request.
* If one of the authentication listeners attempts to perform authentication and it succeeds then the firewall's job is done and it skips the rest of its listeners.
* If one of the authentication listeners attempts to perform authentication and it fails, no more authentication attempts will occur. Whichever authentication failure handler is configured to handle failure for that listener will take over. Usually this means there is a bug in the application or that the credentials on the request were incomplete/incorrect.
* Remember that not all requests require authentication. For example, in most cases somebody does not need to be authenticated to view an "about us" page.

So if you read the above you will noticed this means exactly one or zero authentication listeners will ever actually be used as the first one too match the request and attempt authentication will be the only one used.

Regardless of which listener was used, the end result of a successful authentication should be an authentication token that is then stored in TokenStorage:

* A token is anything that implements `Symfony\Component\Security\Core\Authentication\Token\TokenInterface`
* The token storage class is `Symfony\Component\Security\Core\Authentication\Token\Storage\TokenStorage`
* Token storage service: security.token_storage
* The token contains the known information about who or what is performing the current request. It may contain the exact user or it may simply say that the whoever is performing the request is an anonymous user or some other information.
* Anonymous Token Class: `Symfony\Component\Security\Core\Authentication\Token\AnonymousToken`

Once the token has been created and stored in TokenStorage the authentication process is done.

### firewall !== Symfony\Component\Security\Http\Firewall 

You may have noticed that in the security.yml configuration file a section like this:
```yml
security:
	#...
    firewalls:
```

Each firewall configured in this section is not actually a separate firewall object. (There should only ever be one `Symfony\Component\Security\Http\Firewall` object registered in the container). What each of those firewall configurations actually does is create and register some number of authentication listeners in the singular firewall object registered in the container. As to be expected, each of those authentication listeners will only act on requests you configured them to match and perform authentication in the manner you configured them to. At the end of the day, it's not a huge distinction, but helpful for understanding what the security component is doing behind the scenes should you ever need to know. If you are interested in seeing how Symfony goes from the configuration in securtiy.yml to registering the AuthenticationListeners with the firewall, you can see the code that does it [here](https://github.com/symfony/symfony/blob/master/src/Symfony/Bundle/SecurityBundle/DependencyInjection/SecurityExtension.php).

## Authorization:

### What is authorization?

Authorization is simply the process of restricting access to certain resources inside your Symfony application.

### What is a resource (in the context of Symfony security authorization)?

A resource is literally anything you want to restrict access to. Common examples are endpoints, objects, actions, data. Its a very vague term. Best to just think of it as "Something the user might want to see or act on.". Some resources don't require any authorization because everybody is allowed to see or use them. For the ones that should not be available to everyone, we restrict access to them via authorization.

### What does it mean to be granted authorization?

Being granted authorization means whoever or whatever is is making the current request is allowed to do or see something. Usually it is a user who is attempting to be authorized, but not always. In reality, the only thing required to perform authorization is a authenticated token in TokenStorage. This token may or may have a user stored on it or a user associated with it in some other way, but it is not required. For example, maybe there is something that only Anonymously Authenticated users should have access to (such as the log in page or registration apge). In this case, authorization would still be performed even though there is no current "user", just the token. Regardless, being granted authorization simply means that the currently authenticated token is allowed to see or do whatever thing it is requesting authorization for.

### So how does symfony do authorization?

Well you can think of it as 3 layers really. At the top there is the AuthorizationChecker object, in the middle there is the AccessDecisionManager object, and at the bottom are a bunch of Voter objects.

So lets start at the top with the AuthorizationChecker object. It is where the entire authorization process begins. Any time Symfony checks if something is allowed or not for the current user, a call to. `\Symfony\Component\Security\Core\Authorization\AuthorizationChecker::isGranted`. It's the only thing you should ever think to look call upon when somewhere in your application you need to check if something is authorized.

The authorization checker doesn't do much itself. Actually, all it does is check to see if there is an authentication token present in TokenStorage (since authorization doesn't make any sense without having already done authentication, we wouldn't even know who the user we are trying to determine authorization for is!). After verifying authentication has actually taken place, it simply delegates the actual job of determining whether or not to grant or deny authorization to the AccessDecisionManager object.

The AccessDecisionManager's job is aggregate all the various different authorization rules (which take the form of Voter objects) in one place and determine the Voter (or Voters) that are pertinent to the "resource" which is currently the current user is attempting to be authorized to access. Most of the time only be a single voter with actually cast a vote when checking authorization, but sometimes there will be multiple or even none. In the case that there are multiple voters, or no voters, for a single authorization check, it is the `AccessDecisionMaker`'s responsibity for deciding how to proceed. How that is depends entirely on how you configured it.
    * AccessDecisionManager::STRATEGY_AFFIRMATIVE - As long as one voter says yes, then grant access no matter what any other voters said.
    * AccessDecisionManager::STRATEGY_CONSENSUS - The majority of voters that actually voted must have said yes, otherwise access will be denied.
    * AccessDecisionManager::STRATEGY_UNANIMOUS - All voters that actually voted must have said yes, otherwise access will be denied.
The case for no voters voting, or for tied votes is also configurable via constructor arguments for AccessDecisionManager. By default access is denied in both of those situations.

So lastly at the very bottom of the process are all the Voter objects. Voter objects are dead simple. They have one method, vote, which returns one of three options:
    * VoterInterface::ACCESS_GRANTED - Yes, do grant access for the current authorization check.
    * VoterInterface::ACCESS_DENIED - No, deny access for the current authorization check.
    * VoterInterface::ACCESS_ABSTAIN - This voter does not contain any logic/rules for determining if the current authorization check should be denied or granted.

There are three parameters of the VoterInterface::vote method and at first glance it might not be entirely clear what each is:
    * $token - The token which was created during the authentication process.
    * $subject - If there is a specific object authorization is being checked for, this is it. There are plenty of times where there is no object, in which case the subject is null. For example, checking if the current user is granted admin access does not involve a specific object.
    * $attributes - Like resource, what attributes actually are is somewhat vague. You can think of them as a list of names.
        * When checking authorization on a specific $subject, the attributes may be the names of the specific actions the current user is requesting authorization to perform on the object. For example, when updating and publishing an article the article would be the $subject and the $attributes might look like `['ARTICLE_PUBLISH', 'ARTICLE_EDIT']`.
        * When checking authorization with no subject, the attributes may be general access levels the system is checking if the current user has. For example, checking if a user is allowed to create a new user the attributes might look something like this ['ROLE_USER_CREATOR']

A very good real example of a voter that doesn't need a subject is the RoleVoter that ships with symfony. It will vote any time at least one of the attributes starts with 'ROLE_', and all it does when casting its vote is check if any of the roles on the current user/authentication token match the attribute that started with 'ROLE_'. You can see the code for the RoleVoter [here](https://github.com/symfony/symfony/blob/master/src/Symfony/Component/Security/Core/Authorization/Voter/RoleVoter.php). 

Simple Example Scenario:

* A simple example of this is a regular user trying to access an endpoint that is restricted to admins.
* Authorization is the process of retrieving the token created during authentication from token storage. If the token is an anonymous user token, it will deny access immediately. 
* If there is a user stored on that token, authorization will proceed to verify that the user meets the required criteria to access the admin endpoint (or if that information is on the token itself, it may get it from the token itself).
* This criteria may be as simple as verifying the user the required role(s) (ROLE_ADMIN).
* The criteria can be something more complicated too, maybe the user has to be a certain age.
* The AuthorizationChecker provides you with the ability define what that criteria is in many different ways. You can do it by configuration in a yml file or by writing a custom class that contains the logic for allowing or denying access to a specific resource.


* Actual Examples:
    * Configuration based authorization rules:
        ```
        # security.yml
        security
            access_control:
                - { path: ^/api/login$, role: IS_AUTHENTICATED_ANONYMOUSLY }
        ```
    * Alternatively you can write your authorization rules as php code, which provides a few benefits.
        * You have unlimited flexibility in defining your authorization rules. There are only so many things you can do with the authorization rules you can define in the configuration files.
        * You can unit test the logic for your authorization rules.
        * Code can more clearly express the exact authorization behavior you desire for each different protected type of resource.
        * For an example of how to write authorization rules in php, look [here](http://symfony.com/doc/current/cookbook/security/voters.html)

No matter how you define your authorization rules, the end result is that the AuthorizationChecker class is provided with these rules, and either through manual calls to AuthorizationChecker::isGranted() in your application code or through event listeners in symfony which act based on your configuration.


### Misc

Below are a few more security related services and types to be aware of, although not explicitly requires, they still are used by the security component.


### User Providers:
* A class that takes a "username" and loads the information for that user, returns a user object for that username.
* Might be from database
* Might be from filesystem
* Might be from an external user directory over ldap or some other external service that provides information about users.
### UserPasswordEncoder:
* UserPasswordEncoders: What their name says they are. Just a class which knows how to encode/hash/encrypt the password of a specific User type. 


### End

Thats it, more or less. There are still some areas of the security component not touched upon here, but the real meat of what happens inside the Symfony security component is covered above.
