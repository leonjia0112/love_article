# Hello World!sss

*C*
```c
#include
int main (void)
{
   puts "Hello, World!"
}
```
*C++*
```c++
#include <iostream>
 
int main()
{
    std::cout << "Hello, world!\n";
}
```
*C#*
```csharp
using System;
namespace HelloWorld
{
    class Hello 
    {
        static void Main() 
        {
            Console.WriteLine("Hello World!");
            Console.ReadKey(); 
        }
    }
}
```
*Python 2*
```python
print "hello world!"
```
*Python 3*
```python
print("hello world!")
```
*Java*
```java
public class HelloWorld {
 
    public static void main(String[] args) {
        System.out.println("Hello, World!");
    }
 
}
```
*JavaScript*
```javascript
document.write("Hello, World!");
```
*Scheme*
```scheme
(define hello-world
  (lambda ()
    (begin
      (display "Hello, World!")
      (newline))))
(hello-world)
```
*Scala*
```scala
object HelloWorld{
    def main(args: Array[String]):Unit ={
       println("Hello, World")
    }
}
```
*Haskell*
```haskell
main = putStrLn "Hello, World!"
```
*Rust*
```rust
fn main(){
     println!("Hello, World!");
}
```
*Ruby*
```ruby
puts 'Hello, world!'
```
*Go*
```go
import "fmt"
func main(){
   fmt.Println("Hello, World!")
}
```
*PHP*
```php
<?php
echo "Hello, World!"
?>
```
*Swift*

# Useful Link
## Unix
[Introduction to text manipulation on UNIX-based systems](https://www.ibm.com/developerworks/aix/library/au-unixtext/index.html)

## Code Editor
[Vim Cheat Sheet](https://devhints.io/vim)

## Git
### Git Basics
[Adding more changes to your last commit](https://medium.com/@igor_marques/git-basics-adding-more-changes-to-your-last-commit-1629344cb9a8)
[How to Git PR From The Command Line](https://hackernoon.com/how-to-git-pr-from-the-command-line-a5b204a57ab1)
[How to write a git commit message](https://chris.beams.io/posts/git-commit/)

## Shell
[pushd and popd example](./pushd_and_popd_example.md)

## Vim
#### Vim Basic
[Converting tabs to spaces](http://vim.wikia.com/wiki/Converting_tabs_to_spaces)

#### Useful Link
[Install the Vim 8.0 and YouCompleteMe with Make on CentOS 7.4](https://medium.com/@chusiang/install-the-vim-8-0-and-youcompleteme-with-make-on-centos-7-4-1573ad780953)

[Vim 8.0 Released, the First Major Update Since 2006](https://www.linuxbabe.com/vim/install-vim-8-0-debian-ubuntu-linux-mint-fedora-centos-arch-linux)

## Python
[Python Decorator](https://www.programiz.com/python-programming/decorator)

[PyFormat](https://pyformat.info/)

## Ansible
[Understand Ansible Role and how does it work](https://docs.ansible.com/ansible/latest/user_guide/playbooks_reuse_roles.html)
[Ansible Modules - shell vs. command](https://blog.confirm.ch/ansible-modules-shell-vs-command/)

## Linux Admin
[Understanding Systemd Units and Unit Files](https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files)

[FEDORA 22: SETTING A STATIC IP ADDRESS](https://danielgibbs.co.uk/2015/07/fedora-22-setting-a-static-ip-address/)

Note:
```
# chattr +i /etc/resolv.conf
```

[RHEL Boot Process](./RHEL_boot_process.md)

## GPG
[Secure email: Encrypt and sign your emails with PGP/GnuPG](https://bjornjohansen.no/secure-email)
## SSH
[How to Set up an SSH Server on a Home Computer](https://dev.to/zduey/how-to-set-up-an-ssh-server-on-a-home-computer)

[SSH Passwordless Login Using SSH Keygen in 5 Easy Steps](https://www.tecmint.com/ssh-passwordless-login-using-ssh-keygen-in-5-easy-steps/)

## Virtual Machine
[How to Manage KVM Virtual Environment using Commandline Tools in Linux](https://www.tecmint.com/kvm-management-tools-to-manage-virtual-machines/)

[How to Install Linux KVM and Create Guest VM with Examples](https://www.thegeekstuff.com/2014/10/linux-kvm-create-guest-vm/)


## OS installation
[How to Install Fedora 29 Alongside With Windows 10 or 8 in Dual-Boot](https://www.tecmint.com/install-fedora-27-with-windows-10-or-8-in-dual-boot/)

## RPM
[RPM Packaging Guide](https://rpm-packaging-guide.github.io/#preparing-source-code-for-packaging)

[Macros: Helpful Shorthand for Package Builders](http://ftp.rpm.org/max-rpm/s1-rpm-inside-macros.html)

## Firefox
[Hide Tab Bar w/ Vertical Tab](https://medium.com/@Aenon/firefox-hide-native-tabs-and-titlebar-f0b00bdbb88b)
