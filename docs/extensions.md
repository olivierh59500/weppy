Extensions
==========

weppy extensions extend the functionality of weppy in various different ways.

Extensions are listed on the [Extension Registry](#) and can be downloaded with
`easy_install` or `pip`. When adding extensions to your application, it is a
good habit to declare them as dependencies in your *requirements.txt* or *setup.py*
file: this way, they can be installed with a simple command or when your application installs.

Using extensions
----------------

An extension typically has accompanying documentation that shows how to use it
correctly. In general, weppy extensions should be named with the format `weppy-foo`
and have a package-name like `weppy_foo`, replacing foo with the desired name.
If the extension is written according to the suggested pattern, using it in your 
application will be quite easy:

```python
from weppy import App
from weppy_foo import Foo

app = App(__name__)

# configure the extension
app.config.Foo.someparam = "something"
# add the extension to our app
app.use_extension(Foo)
# access extension attributes and methods
app.ext.Foo.bar()
```

That's all.   

As you can see, extensions have a *namespace* that accesses your app's configuration,
and after you have added the extension to your application using the `use_extension()`
method, you can access the extension instance at `app.ext.<extension_name>`.


Building extensions
-------------------

The first step in creating a new extension for weppy is writing an `Extension` subclass:

```python
from weppy.extensions import Extension

class Awesomeness(Extension):
    default_config = {}

    def on_load(self):
        # pass
```

As you can see, the `Extension` class in actually quite simple, since you just have to write down the default configuration (if needed) and override the `on_load` method, that will be called by the framework when the extension will be initialised.

You can access three attributes of the extension instance, that are injected by weppy before calling the `on_load` method, in particular you will have:

| attribute | description |
| --- | --- |
| app | the application on which the extension is loaded |
| config | a `sdict` containing the configuration |
| env | a `sdict` reserved to the extension where data can be stored |

The `config` attribute will contain the configuration defined by the developer using the extension, with the default attributes you defined in the `default_config` if not specified differently.

The developer will access `app.config.Awesomeness` in order configure the extension, in fact weppy uses the name of your class as namespace for the configuration and environment objects. If you want to specify a different namespace, you can use the relevant attribute:

```python
class Awesomeness(Extension):
    namespace = "Foobar"
```

just remember to update the documentation of your extension accordingly.

Since the extension can access the application object, it can easily do anything, for example it may define a route:

```python
class Awesomeness(Extension):
    def on_load(self):
        self.app.route('/awesome')(awesome_route)

def awesome_route():
    return {'message': 'Awesome!'}
```

or add a custom pipe:

```python
class AwesomePipe(Pipe):
    # some code
    
class Awesomeness(Extension):
    def on_load(self):
        self.app.pipeline.append(
            AwesomePipe()
        )
```

> – dude, what if I need to access the database?  
> – *ask to the developer to add it to the extension configuration*   

In fact, if you state you need the database in the extension documentation, the developer can just write:

```python
app.config.Awesomeness.db = db
```

Then you can write a check inside your load method in order to ensure the database presence:

```python
class Awesomeness(Extension):
    def on_load(self):
        if not self.config.db:
            raise RuntimeError('Awesomeness extension needs the database')
```

### Template extensions

Whenever you want to extend something related to the weppy templating system, you can take advantage of another class provided by the framework: the `TemplateExtension` one.

This class provides some useful methods when dealing with templates, and has to be used on conjunction with the `Extension` one. Let's see this with an example.

Let's say we want to build an extension that adds [Haml](http://weppy.org/extensions/haml) support to weppy. Then we need to write a template extension that interact with templates with an *.haml* file extension and that provides the compiled html source in order to let weppy understand the templates. We can start by writing:

```python
from weppy.extensions import TemplateExtension

class HamlTemplates(TemplateExtension):
    file_extension = '.haml' 
```

Then we can use three different methods provided by the `TemplateExtension` class:

- the `preload` method, that should accept a path and a filename and return the same tuple, and is useful to alter the standard template names weppy looks for;
- the `preprocess` method, that should accept the source code and file name variables and return the source code that should be used by weppy;
- the `inject` method, that should accept a context dictionary and can add methods and variable to it.

Let's say we want to compile the haml templates in html ones on application start, then just tell weppy to use the generated html files one. The simplest way to do that is to override the `preload` method in order to change the extension of the file:

```python
from weppy.extensions import TemplateExtension

class HamlTemplates(TemplateExtension):
    file_extension = '.haml'
    
    def preload(self, path, file_name):
        # file_name will be like "somefile.haml"
        return path, file_name + ".html" 
```

Then we should write the code that compiles the templates:

```python
import os

class Haml(Extension):
    def on_load(self):
        for path, dirs, files in os.walk(self.app.template_path):
            for fname in files:
                if os.path.splitext(fname)[1] == ".haml":
                    self.compile(path, fname)
```

where `compile` method will be the one responsible to parse the haml code and produce compatible html for weppy.

Given this, we also want to register the template extension when the user activate the main extension, and to share the same context, so that the final code will look like this:

```python
class Haml(Extension):
    def on_load(self):
        self.app.add_template_extension(HamlTemplates)
        for path, dirs, files in os.walk(self.app.template_path):
            for fname in files:
                if os.path.splitext(fname)[1] == ".haml":
                    self.compile(path, fname)


class HamlTemplates(TemplateExtension):
    namespace = 'Haml'
    file_extension = '.haml'
    
    def preload(self, path, file_name):
        # file_name will be like "somefile.haml"
        return path, file_name + ".html" 
```

Sharing the namespace between the `Extension` and the `TemplateExtension` is useful, because you will have the same `config` object for the two, and you can pass variables between them using the `env` object:

```python
class Haml(Extension):
    def on_load(self):
        self.env.debug = True


class HamlTemplates(TemplateExtension):
    namespace = 'Haml'
    
    def preload(self, path, file_name):
        if self.env.debug:
            # some code
```

Template extensions can also register *lexers*, which are the keyword used by weppy in templates to render specific contents. For example, the standard `include_static` keyword is a lexer that produce the appropriate `<link>` or `<script>` html objects.

In order to create a new lexer, you have to use the `TemplateLexer` class provided by weppy. Let's say we want to create a shortcut to include images from the static folder using this notation:

```html
<div>
    {{img 'foo.png'}}
</div>
```

To do this, we first need a method that produce the final html code and add it to the template context so we can invoke it:

```python
from weppy import url

class ImgTemplateExtension(TemplateExtension):
    def gen_img_string(self, name):
        return '<img src="{}" />'.format(
            url("static", "images/" + name)
        )
    
    def inject(self, context):
        context['_img_lexer_'] = self.gen_img_string
```

then we should write a lexer that converts the *img* notation to a call to our method and add it as a template *node* to the template tree:

```python
from weppy.extensions import TemplateLexer

class ImgLexer(TemplateLexer):
    def process(self, value):
        s = '_img_lexer_("{}")'.format(value)
        node = self.parser.create_node(
            s, pre_extend=False, writer_escape=False
        )
        self.top.append(node)
```

The above code tells the template parser to add a python node to the current template tree, so that the `_img_lexer_` method will be invoked. The `self.top` property is the actual level of the template tree, so if you are, for example, inside a div or a specific block, that will be used to inject the code.

The last things we need is to register the lexer and the template extension, so the final code will look like this:

```python
from weppy import url
from weppy.extensions import Extension, TemplateExtension, TemplateLexer

class ImgExtension(Extension):
    namespace = "TemplateImg"
    
    def on_load(self):
        self.app.add_template_extension(ImgTemplateExtension)

class ImgLexer(TemplateLexer):
    def process(self, value):
        s = '_img_lexer_("{}")'.format(value)
        node = self.parser.create_node(
            s, pre_extend=False, writer_escape=False
        )
        self.top.append(node)
        

class ImgTemplateExtension(TemplateExtension):
    namespace = "TemplateImg"
    lexers = {'img': ImgLexer}

    def gen_img_string(self, name):
        return '<img src="{}" />'.format(
            url("static", "images/" + name)
        )
    
    def inject(self, context):
        context['_img_lexer_'] = self.gen_img_string
```
