# Решение ДЗ к занятию «2.4. Инструменты Git»

1. Код для решения: 
```
$ git show --pretty=format:'%H %s' --shortstat aefea
aefead2207ef7e2aa5dc81a34aedf0cad4c32545 Update CHANGELOG.md
 1 file changed, 1 insertion(+)
```
Ответ:
___
Полный хэш: aefead2207ef7e2aa5dc81a34aedf0cad4c32545

Комментарий: Update CHANGELOG.md
___

2. Код для решения:
```
$ git show --pretty=format:'%s' --shortstat 85024d3
v0.12.23
 2 files changed, 2 insertions(+), 2 deletions(-)
``` 
Ответ:
___

Тег: v0.12.23
___

3. Код для решения:
```
$ git show --pretty=format:'%P' --shortstat b8d720
56cd7859e05c36c06b56d013b55a252d0bb7e158 9ea88f22fc6269854151c571162c5bcf958bee2b
 1 file changed, 13 insertions(+), 2 deletions(-)
```
Ответ:
___

2 родителя с хэшами:
56cd7859e05c36c06b56d013b55a252d0bb7e158
9ea88f22fc6269854151c571162c5bcf958bee2b
___

Или можно так:
```
$ git show b8d720^ b8d720^2 --shortstat --pretty=format:'%H'
56cd7859e05c36c06b56d013b55a252d0bb7e158
 2 files changed, 2 insertions(+)

9ea88f22fc6269854151c571162c5bcf958bee2b
 1 file changed, 13 insertions(+), 2 deletions(-)
```
4. Код для решения:
```
$ git log v0.12.23..v0.12.24 --oneline
33ff1c03b (tag: v0.12.24) v0.12.24
b14b74c49 [Website] vmc provider links
3f235065b Update CHANGELOG.md
6ae64e247 registry: Fix panic when server is unreachable
5c619ca1b website: Remove links to the getting started guide's old location
06275647e Update CHANGELOG.md
d5f9411f5 command: Fix bug when using terraform login on Windows
4b6d06cc5 Update CHANGELOG.md
dd01a3507 Update CHANGELOG.md
225466bc3 Cleanup after v0.12.23 release
```

5. Код для решения:
```
$ git log -S 'func providerSource(' --oneline
8c928e835 main: Consult local directories as potential mirrors of providers

$ git grep 'func providerSource(' 8c928e835
8c928e835:provider_source.go:func providerSource(services *disco.Disco) getproviders.Source {
```
Ответ:
___

Коммит, где была определена функция func providerSource(...) - это 8c928e835
___

6. Код для решения:
```
$ git log -S 'func globalPluginDirs(' --oneline
8364383c3 Push plugin discovery down into command package

$ git grep 'func globalPluginDirs(' 8364383c3
8364383c3:plugins.go:func globalPluginDirs() []string {

$ git log -L :globalPluginDirs:plugins.go
commit 78b12205587fe839f10d946ea3fdc06719decb05
Author: Pam Selle <204372+pselle@users.noreply.github.com>
Date:   Mon Jan 13 16:50:05 2020 -0500

    Remove config.go and update things using its aliases

diff --git a/plugins.go b/plugins.go
--- a/plugins.go
+++ b/plugins.go
@@ -16,14 +18,14 @@
 func globalPluginDirs() []string {
        var ret []string
        // Look in ~/.terraform.d/plugins/ , or its equivalent on non-UNIX
-       dir, err := ConfigDir()
+       dir, err := cliconfig.ConfigDir()
        if err != nil {
                log.Printf("[ERROR] Error finding global config directory: %s", err)
        } else {
                machineDir := fmt.Sprintf("%s_%s", runtime.GOOS, runtime.GOARCH)
                ret = append(ret, filepath.Join(dir, "plugins"))
                ret = append(ret, filepath.Join(dir, "plugins", machineDir))
        }

        return ret
 }

commit 52dbf94834cb970b510f2fba853a5b49ad9b1a46
Author: James Bardin <j.bardin@gmail.com>
Date:   Wed Aug 9 17:46:49 2017 -0400

    keep .terraform.d/plugins for discovery

diff --git a/plugins.go b/plugins.go
--- a/plugins.go
+++ b/plugins.go
@@ -16,13 +16,14 @@
 func globalPluginDirs() []string {
        var ret []string
        // Look in ~/.terraform.d/plugins/ , or its equivalent on non-UNIX
        dir, err := ConfigDir()
        if err != nil {
                log.Printf("[ERROR] Error finding global config directory: %s", err)
        } else {
                machineDir := fmt.Sprintf("%s_%s", runtime.GOOS, runtime.GOARCH)
+               ret = append(ret, filepath.Join(dir, "plugins"))
                ret = append(ret, filepath.Join(dir, "plugins", machineDir))
        }

        return ret
 }

commit 41ab0aef7a0fe030e84018973a64135b11abcd70
Author: James Bardin <j.bardin@gmail.com>
Date:   Wed Aug 9 10:34:11 2017 -0400

    Add missing OS_ARCH dir to global plugin paths

    When the global directory was added, the discovery system still
    attempted to search for OS_ARCH subdirectories. It has since been
    changed only search explicit paths.

diff --git a/plugins.go b/plugins.go
--- a/plugins.go
+++ b/plugins.go
@@ -14,12 +16,13 @@
 func globalPluginDirs() []string {
        var ret []string
        // Look in ~/.terraform.d/plugins/ , or its equivalent on non-UNIX
        dir, err := ConfigDir()
        if err != nil {
                log.Printf("[ERROR] Error finding global config directory: %s", err)
        } else {
-               ret = append(ret, filepath.Join(dir, "plugins"))
+               machineDir := fmt.Sprintf("%s_%s", runtime.GOOS, runtime.GOARCH)
+               ret = append(ret, filepath.Join(dir, "plugins", machineDir))
        }

        return ret
 }

commit 66ebff90cdfaa6938f26f908c7ebad8d547fea17
Author: James Bardin <j.bardin@gmail.com>
Date:   Wed May 3 22:24:51 2017 -0400

    move some more plugin search path logic to command

    Make less to change when we remove the old search path

diff --git a/plugins.go b/plugins.go
--- a/plugins.go
+++ b/plugins.go
@@ -16,22 +14,12 @@
 func globalPluginDirs() []string {
        var ret []string
-
-       // Look in the same directory as the Terraform executable.
-       // If found, this replaces what we found in the config path.
-       exePath, err := osext.Executable()
-       if err != nil {
-               log.Printf("[ERROR] Error discovering exe directory: %s", err)
-       } else {
-               ret = append(ret, filepath.Dir(exePath))
-       }
-
        // Look in ~/.terraform.d/plugins/ , or its equivalent on non-UNIX
        dir, err := ConfigDir()
        if err != nil {
                log.Printf("[ERROR] Error finding global config directory: %s", err)
        } else {
                ret = append(ret, filepath.Join(dir, "plugins"))
        }

        return ret
 }

commit 8364383c359a6b738a436d1b7745ccdce178df47
Author: Martin Atkins <mart@degeneration.co.uk>
Date:   Thu Apr 13 18:05:58 2017 -0700

    Push plugin discovery down into command package

    Previously we did plugin discovery in the main package, but as we move
    towards versioned plugins we need more information available in order to
    resolve plugins, so we move this responsibility into the command package
    itself.

    For the moment this is just preserving the existing behavior as long as
    there are only internal and unversioned plugins present. This is the
    final state for provisioners in 0.10, since we don't want to support
    versioned provisioners yet. For providers this is just a checkpoint along
    the way, since further work is required to apply version constraints from
    configuration and support additional plugin search directories.

    The automatic plugin discovery behavior is not desirable for tests because
    we want to mock the plugins there, so we add a new backdoor for the tests
    to use to skip the plugin discovery and just provide their own mock
    implementations. Most of this diff is thus noisy rework of the tests to
    use this new mechanism.

diff --git a/plugins.go b/plugins.go
--- /dev/null
+++ b/plugins.go
@@ -0,0 +16,22 @@
+func globalPluginDirs() []string {
+       var ret []string
+
+       // Look in the same directory as the Terraform executable.
+       // If found, this replaces what we found in the config path.
+       exePath, err := osext.Executable()
+       if err != nil {
+               log.Printf("[ERROR] Error discovering exe directory: %s", err)
+       } else {
+               ret = append(ret, filepath.Dir(exePath))
+       }
+
+       // Look in ~/.terraform.d/plugins/ , or its equivalent on non-UNIX
+       dir, err := ConfigDir()
+       if err != nil {
+               log.Printf("[ERROR] Error finding global config directory: %s", err)
+       } else {
+               ret = append(ret, filepath.Join(dir, "plugins"))
+       }
+
+       return ret
+}
```
Ответ:
___

Таким образом имеем 5 коммитов, где была изменена функция globalPluginDirs в файле plugins.go:

commit 78b12205587fe839f10d946ea3fdc06719decb05

commit 52dbf94834cb970b510f2fba853a5b49ad9b1a46

commit 41ab0aef7a0fe030e84018973a64135b11abcd70

commit 66ebff90cdfaa6938f26f908c7ebad8d547fea17

commit 8364383c359a6b738a436d1b7745ccdce178df47
___

7. Код для решения:
```
$ git log -S 'func synchronizedWriters('
commit bdfea50cc85161dea41be0fe3381fd98731ff786
Author: James Bardin <j.bardin@gmail.com>
Date:   Mon Nov 30 18:02:04 2020 -0500

    remove unused

commit 5ac311e2a91e381e2f52234668b49ba670aa0fe5
Author: Martin Atkins <mart@degeneration.co.uk>
Date:   Wed May 3 16:25:41 2017 -0700

    main: synchronize writes to VT100-faker on Windows

    We use a third-party library "colorable" to translate VT100 color
    sequences into Windows console attribute-setting calls when Terraform is
    running on Windows.

    colorable is not concurrency-safe for multiple writes to the same console,
    because it writes to the console one character at a time and so two
    concurrent writers get their characters interleaved, creating unreadable
    garble.

    Here we wrap around it a synchronization mechanism to ensure that there
    can be only one Write call outstanding across both stderr and stdout,
    mimicking the usual behavior we expect (when stderr/stdout are a normal
    file handle) of each Write being completed atomically.   
```
Ответ:
___

Автор функции synchronizedWriters - Martin Atkins <mart@degeneration.co.uk>
___
