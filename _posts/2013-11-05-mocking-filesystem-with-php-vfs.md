---
layout: post
title: "Mocking filesystem with php-vfs"
description: "How to mock filesystem while unit testing."
category: "Testing"
tags: [mocking,filesystem,vfs,unit,testing]
---
{% include JB/setup %}


Often times when developing I came across a situation where there is some sort of filesystem (later 'fs') functionality
required. Be it cache generation, reports, compiling configuration or user directory creation, there will always be
code responsible for fs operations and this code should be tested as well as any other part of the system.

The problem with PHP is that pretty much every fs related function is placed within the low level API and can't be easily
mocked and injected into your test subject as dependency. Ever since PHP 4 there has been quite good support for stream
wrappers and we have the ability to create our own - this ability is exactly what allows us to mock filesystem in a way that can
be easily used within unit test.

You may ask why even bother if you can simply use fixtures or temporary directories? Well, the answer is simple; because
using underlying fs creates dependency on that fs and a unit test with dependency isn't really a unit, is it? The fact is
that there will always be questions and things going wrong when using real fs. Do you have permissions? What
if some other parallel test modifies the fixture? What if the test fails to complete and we never clears temporary files?

Answers to above triggered me to start looking for a different alternative, one where I don't have to think
about the environment, where I don't have to worry about the configuration or permissions, or whether I'm on unix or windows.

This is how I first encountered the vfsStream implementation by bovigo. The idea was great but the execution, I thought, a bit dated.
The wrapper in vfsStream is registered via static method and is global to the process, package interfaces are all over the
place and the whole thing has somewhat PHP4-esque feel.

I decided to deliver something that will offer the same if not more functionality, be structured better and thus easier to adapt - enter php-vfs.

Let's assume we have built a CMS system and we want to provide a setup process to make installation easier. In most cases what you find in these installers is an
interface/page where file permissions are checked against what is required by the application. What you may want to check are things like whether cache and log dirs
are writable, whether you have read access to config files and so on.

Below is a very simple class that prints √ or X based on check result - the idea is that you run it as the first step of the installer.

{% highlight php linenos %}<?php
class Checker {

    protected $root;

    public function __construct($root)
    {
        $this->root = $root;
    }

    public function result($result, $header)
    {
        echo $header;
        if ($result) {
           echo '√';
        } else {
            echo 'x';
        }
        echo PHP_EOL;
    }

    public function checkCache()
    {
        $a = is_dir($this->root.'/cache');
        $b = is_writable($this->root.'/cache');
        return is_dir($this->root.'/cache') && is_writable($this->root.'/cache');
    }

    public function checkLog()
    {
        return is_dir($this->root.'/logs') && is_writable($this->root.'/logs');
    }

    public function checkLib()
    {
        return  !is_writable($this->root.'/lib') && is_readable($this->root.'/lib');
    }

    public function checkInstaller()
    {
        return !file_exists($this->root.'/lib');
    }

}
{% endhighlight %}

Pretty straight forward. It takes configurable APP_ROOT as a constructor parameter and then checks whether certain folders
are accessible to the application.

Here is how it could be used during the process:

{% highlight php linenos %}<?php
require_once 'Checker.php';

define('APP_DIR', __DIR__);

echo 'Checking filesystem permissions:'.PHP_EOL;
$checker = new Checker(APP_DIR);
$checker->result($checker->checkCache(), 'Cache: ');
$checker->result($checker->checkLib(), 'Lib: ');
$checker->result($checker->checkLog(), 'Log: ');
$checker->result($checker->checkInstaller(), 'Installer removed: ');
{% endhighlight %}

Which would produce following result when run from the command line:

    Checking filesystem permissions:
    Cache: x
    Lib: x
    Log: x
    Installer removed: √

Now, how can you actually verify that your code works as expected? By writing unit test! Obviously!

Here is a classic take on a unit test (I will test one method only for clarity) in given situation:

{% highlight php linenos %}<?php
class CheckerTest extends PHPUnit_Framework_TestCase {

    public function testCheckingForCacheReturnsWritableState()
    {
        mkdir($root = '/tmp/'.uniqid());

        $checker = new Checker($root);

        $this->assertFalse($checker->checkCache());

        mkdir($cache = $root.'/cache');

        chmod($cache, 0000);
        $this->assertFalse($checker->checkCache());

        chmod($cache, 0700);

        $this->assertTrue($checker->checkCache());

        rmdir($cache);
        rmdir($root);
    }
}
{% endhighlight %}

Pretty good, we've managed to test and prove that our class actually does what we expect it to do. While above works, there are some
strings attached to that test. The obvious one is that when we run our test during development and it fails, the temporary directory, created
near the top of our test, will never get removed. Other considerations include file permissions, what if we can't write to ```/tmp``` or what if
we are on Windows and there is no ```/tmp``` at all?

For this exact reason php-vfs was created. We can mock the file system and never have to touch the real deal at all! Here's how it's done:

{% highlight php linenos %}<?php
require_once 'Checker.php';

class CheckerWithMockTest extends PHPUnit_Framework_TestCase {

    public function testCheckingForCacheReturnsWritableState()
    {
        $fs = new \VirtualFileSystem\FileSystem();
        $checker = new Checker($fs->path('/'));

        $this->assertFalse($checker->checkCache());

        $cache = $fs->createDirectory('/cache');

        chmod($fs->path('/cache'), 0000);
        $this->assertFalse($checker->checkCache());

        chmod($fs->path('/cache'), 0700);

        $this->assertTrue($checker->checkCache());
    }
}
{% endhighlight %}

As you can see the dependency on underlying fs has been removed and the unit test can be run in total isolation. We don't
have to worry about cleaning, permissions or whether the ```/tmp``` directory is there at all. Everything is kept in memory
and standard low level fs API is working as expected.

php-vfs can be easily integrated into your development code using composer, you can read more about it at [php-vfs github page](http://thornag.github.io/php-vfs);

I'm working at providing it as a PEAR package for system wide installation.
