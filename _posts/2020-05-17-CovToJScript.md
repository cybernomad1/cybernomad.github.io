---
title: "CovToJScript Automation"
categories:
  - Blog
tags:
  - C#
  - Covenant
  - Red Teaming
---



Automating Gadget2Jscript for Covenant C2.

The final code/project can be found [here](https://github.com/cybernomad1/CovToJScript).

I recently found myself in a lab environment that required escalation via phishing utilising malicious HTA files. While this isn’t typically a problem and there are loads of resources out there to utilise this attack vector in order to get RCE – my C2 of choice, Covenant, is lacking this ability when targeting Windows 10/2016.

![CovenantLaunchers]({{site.url}}/assets/posts/CovToJScript/CovenantLaunchers.png)

After a bit of GoogleFu a potential solution to this was found, [GadgetToJScript]( https://github.com/med0x2e/GadgetToJScript) by Mohamed El Azaar. This tool generates .NET serialized gadgets that can trigger assembly load/execution when deserialized via BinaryFormatter from JScript, VBScript or VBA. Most importantly, the resulting output is compatible with Windows 10/2016.

While this was a great starting point, unfortunately the resulting hta files get flagged by defender. An additional blog post by [3xpl01tc0d3r](https://3xpl01tc0d3r.blogspot.com/2020/02/gadgettojscript-covenant-donut.html) was also identified which did provide a valid method for generating hta files which bypassed defender utilising GadgetToJscript.

1. Generate the binary launcher and click on Download button to download the GruntStager.exe file.
2. Create a new file named payload.txt in the folder where GadgetToJScript.exe file is placed.
3. Copy the APC Queue Process Injection code from [here](https://gist.githubusercontent.com/3xpl01tc0d3r/ecf5e1ac09935c674a9c6939c694da13/raw/238ed3339a458ce0260f98dc18a38fdbed420457/Payload.txt) and paste the code in the payload.txt file.
4. Use Donut and generate the shellcode for the GruntStager.exe file and encode the shellcode into Base64 format and copy the Base64 encoded value to variable b64 in payload.txt file.
5. Execute GadgetToJScript.exe to generate hta/JS etc payload for covenant grunt.

While this certainly isn’t the most long winded method – It can be a bit of a pain. Therefore I decided I’d automate it via the Covenant API.

## API
Covenant comes with a powerful API that can be used to create Covenant extensions, perform automation, gain access to data not presented in the web interface, or debug during development.

The best way to visualise the API is to load it into Swagger UI.

![SwaggerUI1]({{site.url}}/assets/posts/CovToJScript/SwaggerUI1.png)

In order to interact with the various Covenant API functions, authentication to ‘api/users/login’ with valid user credentials is required.

![SwaggerUI2]({{site.url}}/assets/posts/CovToJScript/SwaggerUI2.png)

The below code utilises System.Text.Json to serialise user submitted credentials into JSON format and unserialise the post response in order to obtain the resulting authorization token.

![APIDoStuff]({{site.url}}/assets/posts/CovToJScript/APIDoStuff.png)

![AuthenicateModel]({{site.url}}/assets/posts/CovToJScript/AuthenticateModel.png)

![Authenticate]({{site.url}}/assets/posts/CovToJScript/Authenticate.png)

We can then utilise this authorisation token to connect to the binary grunt API – allowing us to obtain the base64 encoded launcher string.

![SwaggerLauncherAPI]({{site.url}}/assets/posts/CovToJScript/SwaggerLauncherAPI.png)

The below code once again uses System.Text.Json to convert the json response into a class instance we can query as needed for relevant information.

![BinaryLauncherModel]({{site.url}}/assets/posts/CovToJScript/BinaryLauncherModel.png)

![GetBinaryLauncher]({{site.url}}/assets/posts/CovToJScript/GetBinaryLauncher.png)

This effectively ends my contribution to the code. The rest of the automation essentially revolves around interfacing with [CSDonut](https://github.com/n1xbyte/donutCS) (a .Net core instance of Donut) and [GadgetToJScript](https://github.com/med0x2e/GadgetToJScript).

![Workflow]({{site.url}}/assets/posts/CovToJScript/Workflow.png)

## Usage
Usage is very similar to the original GadgetToJscript, with the only new requirements being a valid Covenant username and password along with the admin interface URL.
```
-c, --ConvantURL=VALUE     https://127.0.0.1:7443
-u, --Username=VALUE       Covenant username
-p, --Password=VALUE       Covenant Password
-w, --scriptType=VALUE     js, vbs, vba or hta
-e, --encodeType=VALUE     VBA gadgets encoding: b64 or hex (default set to
                             b64)
-o, --output=VALUE         Generated payload output file, example:
                             C:\Users\userX\Desktop\output (Without extension)
-r, --regfree              registration-free activation of .NET based COM
                            components
-h, --help=VALUE           Show Help
```
![Usage]({{site.url}}/assets/posts/CovToJScript/Usage.png)

Execution of the resulting file will result in a covenant grunt being executed in memory, whilst also bypassing both AMSI and current instances of Defender AV.
