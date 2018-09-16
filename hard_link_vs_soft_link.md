# Hard Link and Symbolic Link

## Definition
#### HardLink
A hard link is merely an additional name for an existing file on Linux or other Unix-like operating systems.

Any number of hard links, and thus any number of names, can be created for any file. Hard links can also be created to other hard links. However, they **cannot be created for directories**, and they **cannot cross filesystem boundaries or span across partitions**.

Perhaps the most useful application for hard links is to allow files, programs and scripts (i.e. short programs) to be easily accessed in a different directory from the original file or executable file (i.e., the ready-to-run version of a program). Typing the name of the hard link will cause the program or script to be executed in the same way as using its original name. (http://www.linfo.org/hard_link.html)

So what does this definition really mean? Well you can create a hard link to an existing file by using the command ln file_name hardlink. I have provided an example below of creating a hard link in action. In the example below I created a hardlink aka a shortcut to the file named file1 with the hardlink named hlink1.
