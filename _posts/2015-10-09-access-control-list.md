---
layout: post
title: "Access Control List"
description: "Controlling access to admin pages using ACL"
category: 
tags: [Acess Control List, Admin Access Control, Recruiter, Python, Django]
---
{% include JB/setup %}


<a href="https://www.hackerearth.com/recruit/" target="_blank">HackerEarth Recruit</a>, is a platform for technical recruitment. Many companies use this platform for candidate assessments and interviewing.
There can be multiple admins for a company account. As teams grow in size, access control is a special concern for applications that deal with financial and privacy data. Access control is concerned with determining the allowed activities of legitimate users, but we required more sophisticated and complex control mediating every attempt by a user to access a resource in the application based on the sensitivity level of various features.

A state of access control is said to be safe if no permission can be leaked to an unauthorized or uninvited principal. Simpliest solution we found was to use Access Control Lists to restric access based on user roles.

### Implementation (Python/Django) ###

  Access Lists can be configured to map roles to access/feature permissions. Each access right should have a unique name, also assign a unique value to each access rights. Please note roles and access rights are used interchangeably in the current context. In the example ACL below, permissions are named after some of the features in Recruit.
  
  The example code below is self explanatory.


*acl.py*

    # defining access rights constants based on the required critria.
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


A utility function which returns roles for any given admin.

*acl.py*

    def get_company_admin_roles(user):
        """ Returns roles for a company admin.
        SuperAdmin has access to all the features/pages,
        so for a superadmin all the roles are returned.
        For admins other than superadmin return only roles assigned.
        """
        roles = []
        admin = user.admin
        
        if admin is not None:
            if SUPERADMIN in admin.roles_list:
                roles = COMPANY_ADMIN_ROLES.keys()
            else:
                roles = admin.roles
        return roles


A decorator is writen to restrict access to the views. The decorator will return 404 if the conditions are not met.

*decorators.py*

    def has_admin_access(role):
        """Decorator to check admin access to features.
        """
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


ACL can also be used in senerios where menu options or page contents are to displayed based on the roles.
To implement access level restrictions at template level, a context processor is written to make the access rights available as variables to the templates. This can also done at view level, but it violates the DRY principle. 

*context_processors.py*

    def company_admin_roles(request):
        """Returns dict containing roles for an admin,
        used in templates to enable admin actions.
        """
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