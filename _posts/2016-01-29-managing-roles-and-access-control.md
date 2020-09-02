---
author: Aishwarya Reddy
layout: post
title: "Managing roles and access control in a web application"
description: "Manage admin roles and restrict access to features."
category: 
tags: [Acess Control List, Admin Access Control, Manage Roles, Recruiter, Python, Django, ACL, HackerEarth]
---
{% include JB/setup %}


<a href="https://www.hackerearth.com/recruit/" target="_blank">HackerEarth Recruit</a>, is a platform for technical recruitment. Many companies use this platform for candidate assessments and interviewing.
There can be multiple admins for a company account. As teams grow in size, access control is a special concern for applications that deal with financial and privacy data. Access control is concerned with determining the allowed activities of legitimate users, but we required more sophisticated and complex control mediating every attempt by a user to access a resource in the application based on the sensitivity level of various features.

**_A state of access control is said to be safe if no permission can be leaked to an unauthorized or uninvited principal._** 

We figured that the simpliest solution to restrict access was to use ACL.


#### What is ACL ? ####
An access control list (ACL), with respect to a computer file system, is a list of permissions attached to an object. An ACL specifies which users or system processes are granted access to objects, as well as what operations are allowed on given objects.


Many kinds of systems implement ACL, or have a historical implementation like Filesystem ACLs and Networking ACLs.

A *filesystem* ACL is a data structure (usually a table) containing entries that specify individual user or group rights to specific system objects such as programs, processes, or files.

In *Networking* ACL refers to rules that are applied to port numbers or IP addresses that are available on a host or other layer 3, each with a list of hosts and/or networks permitted to use the service.


For <a href="https://www.hackerearth.com/recruit/" target="_blank">Recruit</a>, the approach had to be role based access restriction to authorized admins. This implementation of access control mechanism is defined around roles and privileges.


### Implementation (Python/Django) ###

Access control Lists can be configured to map roles to features.
In this ACL implementation, roles are named after existing features which require access control.
Each access right should have a unique name, and also assign a unique value to each.

  
The example code snippets below are self explanatory.

*acl.py*

    # defining account admin roles based on the required critria.
    SUPERADMIN = 1
    TEST_ADMIN = 2
    INTERVIEW_ADMIN = 3
    LIBRARY_ADMIN = 4

    # access permissions are mapped to human readable names.
    COMPANY_ADMIN_ROLES = {
        SUPERADMIN: 'Super Admin',
        TEST_ADMIN: 'Tests Admin',
        INTERVIEW_ADMIN: 'Interviews Admin',
        LIBRARY_ADMIN: 'Library Admin',
    }

    # used as variable names in context processors, explained below.
    MAP_ROLE_ID_NAME = {
        SUPERADMIN : 'SUPERADMIN',
        TEST_ADMIN : 'TEST_ADMIN',
        INTERVIEW_ADMIN : 'INTERVIEW_ADMIN',
        LIBRARY_ADMIN : 'LIBRARY_ADMIN',
    }


*acl.py*

To retrive assigned roles for any given account admin, a utility is written.
If the given admin is a SuperAdmin, all the roles are returned as SuperAdmin has access to all the features.


    def get_company_admin_roles(user):
        roles = []
        admin = user.admin
        
        if admin is not None:
            if SUPERADMIN in admin.roles_list:
                roles = COMPANY_ADMIN_ROLES.keys()
            else:
                roles = admin.roles
        return roles


*decorators.py*

At the view level, access restriction is handled by wrapping views with decorator which checks for access permissions.
The decorator will raise 404 if an admin has no access permission to the feature being accessed.


    def has_admin_access(role):
        def decorator(f):
            @wraps(f)
            @login_required
            def _company_acl(request, *args, **kwargs):

                # Checks an admin is a superuser or admin has
                # permission to access view
                roles = get_company_admin_roles(user)
                if SUPERADMIN in roles or role in roles:
                    return f(request, *args, **kwargs)

                # if admin has no access then raise no access
                raise Http404
            return _company_acl
        return decorator


    views.py

    from acl.py import LIBRARY_ADMIN
    from decorators.py import has_admin_access

    # decorator check before processing the request.

    @has_admin_access(LIBRARY_ADMIN)
    def library(request):

        template = 'library.html'
        ...
        ...



In <a href="https://www.hackerearth.com/recruit/" target="_blank">Recruit</a> app, menu options and page contents are also customized based on account admin roles and hence the need to implement access restriction at template level too. 

This is achieved by writing a **context processor** which makes the account admin roles avaiable as variables to the templates. This can also done at view level, but it violates the DRY principle.

*context_processors.py*

    def company_admin_roles(request):
        return_dict = {}

        admin = request.user.admin
        admin_roles = admin.roles_list

        # if admin is superadmin set all roles to true

        if SUPERADMIN in admin_roles:
            for key, value in MAP_ROLE_ID_NAME.items():
                return_dict.update({value: True})
        else:
            for role in admin_roles:
                return_dict.update({MAP_ROLE_ID_NAME[role]: True})

        return return_dict

In templates the context variables can be used to check the access permissions. Refer to the code below :-

*menu.html*

{% highlight html %}
<ul>
    <li><div class="">HackerEarth</div></li>
    <li><a href="">Home</div></a></li>

    {{"{% if TEST_ADMIN"}} %}
    <li><a href="">Tests</a></li>
    {{"{% endbif"}} %}

    {{"{% if LIBRARY_ADMIN" }} %}
    <li><a href="">Questions Library</div></a></li>
    {{"{% endif" }} %}


</ul>
{% endhighlight %}


*Posted by [Aishwarya Reddy](http://hck.re/areddy).*