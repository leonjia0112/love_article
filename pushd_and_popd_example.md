# `pushd` and `popd` Example

`pushd`, `popd`, and `dirs` are shell builtins which allow you manipulate 
the directory stack. This can be used to change directories but return to 
the directory from which you came.

For example
start up with the following directories:
```
$ ls
dir1  dir2  dir3
```
pushd to dir1
```
$ pushd dir1
~/somedir/dir1 ~/somedir
$ dirs
~/somedir/dir1 ~/somedir
```
dirs command confirms that we have 2 directories on the stack now. dir1 
and the original dir, somedir.

pushd to ../dir3 (because we're inside dir1 now)
```
$ pushd ../dir3
~/somedir/dir3 ~/somedir/dir1 ~/somedir
$ dirs
~/somedir/dir3 ~/somedir/dir1 ~/somedir
$ pwd
/home/saml/somedir/dir3
```
dirs shows we have 3 directories in the stack now. dir3, dir1, and somedir. 
Notice the direction. Every new directory is getting added to the left. 
When we start poping directories off, they'll come from the left as well.

manually change directories to ../dir2
```
$ cd ../dir2
$ pwd
/home/saml/somedir/dir2
```
Now start popping directories
```
$ popd
~/somedir/dir1 ~/somedir
$ pwd
/home/saml/somedir/dir1
```
Notice we popped back to dir1.

Pop again...
```
$ popd
~/somedir    
$ pwd
/home/saml/somedir
```
And we're back where we started, somedir.

Might get a little confusing, but the head of the stack is the directory 
that you're currently in. Hence when we get back to somedir, even though 
dirs shows this:
```
$ dirs
~/somedir
```
Our stack is infact empty.
```
$ popd
bash: popd: directory stack empty
```
