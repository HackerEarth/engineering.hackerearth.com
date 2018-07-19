---
layout: post
title: "What you see is what you get!"
description: "Integration of WYSIWYG editor in HackerEarth's Recruit platform"
category:
tags: [CKEditor, WYSIWYG editor, HackerEarth]
---

### Introduction
HackerEarth has grown into a platform that serves a huge number of customers for technical assessment. To make this possible, we try our best to make the platform as easy-to-use as it can get.

At several places in our Recruiter Dashboard, we used to have a [Markdown](https://en.wikipedia.org/wiki/Markdown) editor to allow users to edit free text. There have been multiple times when many of our recruiters have struggled to create content using the Markdown editor. They need not to worry anymore. After many such requests to improve this, we came up with a fix. Say hello to **CKEditor**—The well-known [WYSIWYG](https://en.wikipedia.org/wiki/WYSIWYG), Rich Text editor.

![Hackerearth - WYSIWYG vs Markdown](/images/wysiwyg_vs_markdown.jpg)

### Why CKEditor?
In the battle of the titans (of WYSIWYG editing) between [CKEditor](https://ckeditor.com/) and [TinyMCE](https://www.tinymce.com/), we decided to go with CKEditor because of the following reasons:
- It has a huge community of active developers. The strength of the community around an open source project is strongly related to the project's success.
- As compared to TinyMCE, it provides better support for the following:
    - Multiple languages
    - Source editing
    - Tables
    - Image and media handling etc.
- It was designed with modularity in mind which allows you to go much deeper if you’re a developer.
- It is doing much better as compared to TinyMCE. One of the easy tricks while surveying software is to compare how alternatives are doing on Google and Stack Overflow trends.

| ![CKEditor Vs TinyMCE](/images/ckeditor_vs_tinymce_1.jpg) |
|:--:|
| *Google search comparison (past 5 years)* |

&nbsp;

| ![CKEditor Vs TinyMCE](/images/ckeditor_vs_tinymce_2.jpg) |
|:--:|
| *Number of Stack Overflow questions asked* |

### Integration
The integration of WYSIWYG editor across HackerEarth’s Recruit platform is broadly divided into three steps:

- **Adding the Django CKEditor package**

As the Recruiter dashboard is written entirely in Django, we decided to integrate CKEditor using the [django-ckeditor](https://github.com/django-ckeditor/django-ckeditor) package. CKEditor provides a huge list of out-of-the-box functionalities. Thinking from the perspective of recruiters and problem setters, we decided to opt for a few of them only. The Django CKEditor package reads the configuration from the `settings.py` file.

Here is the snapshot of what the CKEditor configuration in the code looks like:

```python
# CKEditor UI and plugins configuration
CKEDITOR_CONFIGS = {
    'default': {
        # Toolbar configuration
        # name - Toolbar name
        # items - The buttons enabled in the toolbar
        'toolbar_DefaultToolbarConfig': [
            {
                'name': 'basicstyles',
                'items': ['Bold', 'Italic', 'Underline', 'Strike', 'Subscript',
                          'Superscript', ],
            },
            {
                'name': 'clipboard',
                'items': ['Undo', 'Redo', ],
            },
            {
                'name': 'paragraph',
                'items': ['NumberedList', 'BulletedList', 'Outdent', 'Indent',
                          'HorizontalRule', 'JustifyLeft', 'JustifyCenter',
                          'JustifyRight', 'JustifyBlock', ],
            },
            {
                'name': 'format',
                'items': ['Format', ],
            },
            {
                'name': 'extra',
                'items': ['Link', 'Unlink', 'Blockquote', 'Image', 'Table',
                          'CodeSnippet', 'Mathjax', 'Embed', ],
            },
            {
                'name': 'source',
                'items': ['Maximize', 'Source', ],
            },
        ],

        # This hides the default title provided by CKEditor
        'title': False,

        # Use this toolbar
        'toolbar': 'DefaultToolbarConfig',

        # Which tags to allow in format tab
        'format_tags': 'p;h1;h2',

        # Remove these dialog tabs (semicolon separated dialog:tab)
        'removeDialogTabs': ';'.join([
            'image:advanced',
            'image:Link',
            'link:upload',
            'table:advanced',
            'tableProperties:advanced',
        ]),
        'linkShowTargetTab': False,
        'linkShowAdvancedTab': False,

        # CKEditor height and width settings
        'height': '250px',
        'width': 'auto',
        'forcePasteAsPlainText ': True,

        # Class used inside span to render mathematical formulae using latex
        'mathJaxClass': 'mathjax-latex',

        # Mathjax library link to be used to render mathematical formulae
        'mathJaxLib': 'https://cdnjs.cloudflare.com/ajax/libs/mathjax/2.7.1/MathJax.js?config=TeX-AMS_SVG',

        # Tab = 4 spaces inside the editor
        'tabSpaces': 4,

        # Extra plugins to be used in the editor
        'extraPlugins': ','.join([
            # 'devtools',  # Shows a tooltip in dialog boxes for developers
            'mathjax',  # Used to render mathematical formulae
            'codesnippet',  # Used to add code snippets
            'image2',  # Loads new and better image dialog
            'embed',  # Used for embedding media (YouTube/Slideshare etc)
            'tableresize',  # Used to allow resizing of columns in tables
        ]),
    }
}
```
Rendering the editor in the front-end is super easy. Out of the two widgets provided by the Django CKEditor package (`CKEditorWidget` and `CKEditorUploadingWidget`), we decided to go with `CKEditorUploadingWidget` because we wanted to include the support for uploading files.

Let's suppose, your `models.py` file contains a model named `MyModel` which contains a CharField named `my_field`. To attach the CKEditor to this field, create  a form as stated below and you are good to go.
```python
from ckeditor_uploader.widgets import CKEditorUploadingWidget
from django import forms

class MyForm(forms.ModelForm):
    class Meta:
        model = MyModel
        fields = ('my_field',)
        widgets = {
            'my_field': CKEditorUploadingWidget(attrs={
                'class': 'my-ckeditor-class'
                'id': 'my-ckeditor-id'
            })
        }
```
By default, `CKEditorUploadingWidget` fetches the configuration from `CKEDITOR_CONFIGS['default']` which is defined in `settings.py`. If you want to use different configurations, say for rendering multiple editors, you can define `my_config` in the `settings.py` file instead of `default` and pass it in the widget as follows:
```python
widget = CKEditorUploadingWidget(config_name='my_config')
```
- **Modifying the code of Django CKEditor to suit our needs**
    <br><br>
    - **Supporting custom language for internationalization**
        <br><br>
        We defined a utility function `get_ckeditor_language` to provide the language in which we want to render the CKEditor.
        &nbsp;
        ```python
        from django.conf import settings

        # CKEditor localization mapping
        CKEDITOR_LOCALE_MAP = {
            'en-us': 'en',
            'ja': 'ja',
            'zh': 'zh-cn',
            'fr': 'fr',
            'es': 'es',
            'pt-br': 'pt-br',
            'id': 'id',
        }

        def get_ckeditor_language():
            """ Returns the UI language localization to be used with CKEditor """
            default_language_code = settings.LANGUAGE_CODE
            default_plugin_language = CKEDITOR_LOCALE_MAP.get(default_language_code,
                                                              default_language_code)
            return default_plugin_language
        ```
        In the `settings.py` file:
        ```python
        # The user interface language localization to be used with CKEditor
        CKEDITOR_UI_LANGUAGE_SELECTOR = 'get_ckeditor_language'
        ```
        In the django-ckeditor `ckeditor/widgets.py` file, we modified the `_set_config` method as follows:
        ```python
        from django.utils.module_loading import import_string
        def _set_config(self):
            lang = import_string(getattr(settings, 'CKEDITOR_UI_LANGUAGE_SELECTOR', 'django.utils.translation.get_language'))()
            if lang == 'zh-hans':
                lang = 'zh-cn'
            elif lang == 'zh-hant':
                lang = 'zh'
            self.config['language'] = lang
        ```
    - **Using custom storage method for image upload**
        <br><br>
        While uploading files through `DefaultStorage` which is provided by Django, the query-string authentication is enabled by default. We need to store the URL of the image while uploading it, and therefore, query-string authentication cannot be used in this case. To prevent this, we created a custom storage class named `PublicMediaRootS3BotoStorage` which inherits from the `S3BotoStorage` package.
        In the `settings.py` file:
        <br><br>
        ```python
        CKEDITOR_STORAGE_BACKEND = 'custom_storages/PublicMediaRootS3BotoStorage'
        ```
         In the `ckeditor_uploader/utils.py` file, we added a new method to fetch the new storage:
        ```python
        # Allow for a custom storage backend defined in settings.
        def get_storage_class():
            return import_string(getattr(settings, 'CKEDITOR_STORAGE_BACKEND', 'django.core.files.storage.DefaultStorage'))()
        storage = get_storage_class()
        ```
        We replaced **default_storage** with storage in all the respective files to make it work seamlessly.

- **Migrating the existing problem data to make it compatible with CKEditor**
    <br><br>
    - **Problem**
        <br><br>
        There are approximately 2.5 lakh problems that contain mathematical symbols spread across multiple tables in our database. All the problems had LaTeX code written within `$$` and `$$`.
        CKEditor provides the support for rendering mathematical symbols using the MathJax plugin which reads the LaTeX written between `\(` and `)\` enclosed by a span containing a unique class.
        The class to be used has to be defined in the `settings.py` file as we did above.
        For example
        `<span class="mathjax-latex">\(Z_{i} = P*X(Z_{i-1})+Q\)</span>`
        <br><br>
    - **Solution**
        <br><br>
        We wrote a script that uses RegEx to fetch all the mathematical symbols enclosed within `$$` from a problem and makes them compatible with CKEditor. Running the script on a whopping 2.5 lakh problems took only 15 minutes to complete!

Here is a snapshot of what the editor looks like in the Recruiter dashboard:

![CKEditor Snapshot](/images/ckeditor_snapshot.jpg)

### Last words...
The integration of CKEditor with HackerEarth’s Recruit platform has brought a plethora of new features that make the job of setting problems easier and interesting. On a platform where hundreds of problems are created and reviewed every day, the amount of work required to deploy the editor into production was worth the effort.

*Peace out!*

*Posted by [Himanshu Malhotra](https://www.linkedin.com/in/psycane/)*
