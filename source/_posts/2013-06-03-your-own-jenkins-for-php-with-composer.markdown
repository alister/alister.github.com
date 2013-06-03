---
layout: post
title: "Your own 'Jenkins' for PHP with Composer"
date: 2013-06-03 21:30
comments: true
categories: php continuous-integration jenkins composer
published: true
---

I'm a big fan of CI - testing, code-coverage, mess-detection and coding standards. I like to automate as much as I can as well, becuase a) it's really neat to see tests and code-coverage output going green, and b) I think it genuinely leads to better software.

Setting up a full Jenkins system is a bit overkill for just my home projects though, and I'd also like to see how I'm doing between git-pushes to the main repository at work as well. So, I've put together a system, bult on top of [Composer](http://getcomposer.org/) that lets me do just that.

<!-- more -->

[{% img right /assets/201306composer-update.thumb.png "Some people are trying to get it" "A week's learnings at my current contract" %}](/assets/201306composer-update.png)

This ['personal-ci'](http://github.com/alister/personal-ci) project, also allows for setting up a full CI-configuration as part of the composer.json file, so you can be sure which versions (ie: up to date) of [PHPunit](https://github.com/sebastianbergmann/phpunit/, [PDepend](http://pdepend.org/), [PHPCPD](https://github.com/sebastianbergmann/phpcpd) and the other related tools, are being run. It's not, however, quite as easy as just listing all the PHP-based code-quality projects into the <code>composer.json</code> file.

The problem with just listing them all out, is getting a configuration that works

 * with composer, out of the box, and
 * takes into account different versions of some support libraries being required by different tools

There are two that require a little more work than the others:

 * phpcb - there is a fork by Github covex-nn with Composer support (and avoiding some older PEAR-based libraries)
 * zetacomponents/console-tools - Unfortunately, Covex-nn's repo requires a different version of ZC's console-tools.

So, to get the best of both, requires a bit of a hack, hence the <code>"as 1.9999"</code> part of the file, below.


``` javascript "composer.json"
{
    "name": "alister/personalci-demo",
    "description": "An example CI-style output",
    "license": "CC-BY-SA-3.0",
    "require": {
        "rhumsaa/uuid": "2.*"
    },
    "require-dev": {
        "phpunit/phpunit": ">=3.7",
        "squizlabs/php_codesniffer": ">=1.5.0RC2",
        "pdepend/pdepend": "dev-master",
        "phpmd/phpmd": "dev-master",
        "sebastian/phpcpd": "dev-master",
        "phploc/phploc": "dev-master",
        "covex-nn/phpcb": "dev-fork",
        "zetacomponents/console-tools": "dev-master as 1.9999",
        "mockery/mockery": ">=0.8"
    },
    "autoload": {
        "psr-0": {
            "ProjectName": "src/"
        }
    }
}
```

So, with this, composer.json file, and a 'composer update', you get all these tools installed into the <code>vendor/bin/</code> directory

 * vendor/bin/phpmd -> ../phpmd/phpmd/src/bin/phpmd
 * vendor/bin/phpcpd -> ../sebastian/phpcpd/composer/bin/phpcpd
 * vendor/bin/phpunit -> ../phpunit/phpunit/composer/bin/phpunit
 * vendor/bin/phploc -> ../phploc/phploc/composer/bin/phploc
 * vendor/bin/phpcs -> ../squizlabs/php_codesniffer/scripts/phpcs
 * vendor/bin/pdepend -> ../pdepend/pdepend/src/bin/pdepend
 * vendor/bin/phpcb -> ../covex-nn/phpcb/bin/phpcb

You can put those vendor/bin paths into your test-runner. For full compatibility with Jenkins, I added to, and edited the sample <code>build.xml</code> file from [http://jenkins-php.org/](http://jenkins-php.org/). A snippet of that is below, but the full version is in the [Github Repo](http://github.com/alister/personal-ci/).


``` xml "build.xml (fragment)"
<?xml version="1.0" encoding="UTF-8"?>
<project name="Personal-CI project" default="build">
 <target name="build"
   depends="prepare,composer,lint,phploc,pdepend,phpmd-ci,phpcs-ci,phpcpd,phpunit,phpcb"/>
 <target name="build-parallel"
   depends="prepare,lint,tools-parallel,phpunit,phpcb"/>

  <!-- ... more ... -->

  <target name="composer" description="Installing dependencies">
    <exec executable="wget" failonerror="true">
      <arg value="-nc" />
      <arg value="http://getcomposer.org/composer.phar" />
    </exec>
    <exec executable="php" failonerror="true">
      <arg value="composer.phar" />
      <arg value="install" /><!--
        # Use the lockfile versions only, if it exists (fall back to .json) -->
      <arg value="--dev" /><!-- includes mockery and all the other QA-tools -->
      <arg value="--prefer-dist" /><!--
            ## download/cache a local .zip file for use -->
      <arg value="--no-progress" />
      <!--arg value="- -install-suggests" /-->
      <!--arg value="- -optimize-autoloader" /
            ## we don't usually optimise deployment -->
    </exec>
  </target>

 <target name="lint" description="Perform syntax check of sourcecode files">
  <apply executable="php" failonerror="true">
   <arg value="-l" />

   <fileset dir="${basedir}/">
    <include name="**/*.php" />
    <exclude name="vendor/" /><!-- Don't check in vendors/
        - libraries are expected to be tested independently -->
    <modified /><!-- Only checking modified files -->
   </fileset>

  <!-- ... more ... -->
</project>
```

With that in place, you can run 'ant' to run the whole system and it generates full-code coverage web-pages for all the files it tests, and imports all the other checks into a Code-Browser. Having that shown up via an actual website though, and not just as a file-based URL is very helpful.

So - I have a simple Apache-Vhost that points to a directory. Within that directory I have symlinks to the <code>build/</code> directories of my other local projects - which allows for full AJAXified access to the CodeBrowsers in each one.

## But wait - there's more!

There's not just one way to run the tests in the personal-ci repo, but three!

### ./runTests.sh

Sometimes, its handy to be able to quickly run PHPUnit, editing in one window, and then rerunning - particularly when you are writing tests as you go, and would like to see nearly instant results.

For that, For many years now, I've been writing a little shell-script, ./runTests.sh:

``` bash "runTests.sh" "(cut-down version)"
#!/bin/sh
clear
mkdir -p ./build/logs ./build/coverage/ ./logs/ ./tmp/
TEST="$@"
#VERBOSE="--verbose "
#COVERAGE="--coverage-html=../build/coverage"
COLORS="--colors"
PHPUNIT="vendor/bin/phpunit"

time \
  $PHPUNIT $GROUP $COLORS $VERBOSE $CONF $MORE $COVERAGE $TEST
# if not running in a pipe (eg, 'less'), re-run on 'Enter'
if [ -t 1 ] ; then
  echo ""
  echo ""
  echo -n "press [Enter] to re-run:> "
  read x
  exec $0 $@
fi
```

Again, a slightly more complete version of that is in the [Github Repo](http://github.com/alister/personal-ci/).

### guard / Guardfile

Finally, one more - quite recent addition to my testing arsenal. This one hails from the Ruby world. Included in the repository is a <code>Gemfile</code> (which is broadly speaking a Ruby equivilent of PHP's <code>composer.json</code>), so that you can run <code>bundle install</code> to install (that Gemfile also includes Capistrano, which I use to deploy PHP code to my server). There's a little more on Capistrano on my [phpscaling.com](http://phpscaling.com/2011/11/24/deployment-with-capistrano-the-gotchas/) blog.

With 'Guard' installed, the provided 'Guardfile' sets up a watch for changes to files named like <code>tests/*Test.php</code> - and to some others that are the original source file equivilents to the tests. Should either one change, the matching tests are also run. To start, just run 'guard' (though you might need to run 'bundle exec guard').

If a test fails, then it will remember that, and keep running it when the source file is edited, and saved. When that test once again succeeds, it will then go and re-run the rest of the tests.

Having 'guard' running in one window while you are editing the project in your editor can really speed up testing while your are developing - the tests are run almost as soon as you have finished writing them.

It's a little bit easy to confuse however, and it's crashed on my a few times requiring a manual 'kill' of the process, so it is a tool I'll use only while doing some pretty hard-core test-driven development sessions. It also does not like any output while you are testing (which in PHPUnit would usually just be printed, abeit messing up the flow of the dots on screen) - so for that, I'll just fall back on the runTests.sh script, that I show above.

To make sure that the tests run quickly, I have also added to the repo, a <code>phpunit.xml</code> configuration - one that does almost exactly the same as the default, but without code coverage being generated. With good unit-tests, that are not doing too much (or, at least, that have some long-duration tests being excluded with @group annotations or test-suites), even when Guard runs all the tests, it's just taking a few seconds.

## Thats How I Test

So, there's a few good, and different, options for testing.

 * Ant and a build.xml file runs from the command line - or in Jenkins - to generate full code coverage and other reports listed in the CodeBrowser
 * runTests.sh - a quick shell script that can rerun your tests, and build code coverage HTML output on demand
 * Guard - a Ruby-based system to run tests as soon as you've saved the file.

And having all the QA-tools pulled in with Composer means that you know they are installed, and exactly what version is being run.


Happy testing!
