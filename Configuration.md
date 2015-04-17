Locate the Apache httpd.conf file. This is the webserver's main configuration file and it is likely to be in a directory named "conf"

Include the following line in the config file ...
```
LoadModule helloworld modules/mod_HelloWorld.so
```

Where in the above ...
  * helloworld is the module name. It is equal to the value of the ModuleName constant you will find in mod\_HelloWorld.dpr .
  * modules is the name of the directory that you put your so file. If you put it somewhere different, change accordingly.
  * mod\_HelloWorld.so is the name of the output binary produced by compiling mod\_HelloWorld.dpr . If you project has a different name, change accordingly.

Now also in the configuration file, after LoadModule, add the following configs ...

```
<IfModule helloworld>
  <Location "/hello">
    SetHandler mod_helloworld-handler
  </Location>
</IfModule>

<IfModule helloworld>
  pascali-setconfigroot banana
</IfModule>
```

The above is just an example. Of course you can configure it any way you want.

  * In the above, the < Location> node, says that when the path part of your URL starts with /hello, then this module will be tasked with handling the web request. The SetHandler command matches the value of the HandlerName constant in the  mod\_HelloWorld.dpr . If you are using a different primary handler name, of course, change accordingly.
  * You can define multiple handlers in the dpr. In-source code comments show you how to do it. Of course, define what-ever Apache configuration you need for each of your handlers.
  * The < pascali-setconfigroo> node is an example of how to pass configuration information from the config file to Delphi. These are called Apache directives. The example pascali-setconfigroot directive show here is dealt with in unit uHelloWorldApacheApp.
    * Define whatever directives your application needs, in the style of uHelloWorldApacheApp.
    * Register your directives in the dpr file, as shown by the in-source comments.
    * Configure your directives with data, as required in the Apache config file.