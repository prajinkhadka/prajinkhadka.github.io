# Managing dotfiles with stow. 

Managing dotfiles has always been a mess for me. It gets complicated when I have to set up a completely new machine with my configurations. 

I reckon there are many like me who have a hard time managing, updating dotfiles continuously and without breaking anything.

There are many other ways, and programs specifically available to manage such as WADM, dotty etc but here I am going to use a symlink manager to manage dotfiles efficiently. 


So, **what are dotfiles ?** 
Dotfiles are configuaration files for each program/tool that you have installed and you want to configure in the way you want.
For example, one simple dotfile is the .gitconfig or, .bashrc or .vimrc which sets up configuration for a specific program/tool as specified. 
Mostly they reside in $HOME/.config or in $HOME.

**Why to manage dotfiles ?** 
Well, as a programmer, much of the time is spent writing, reviewing code, ( of course stand ups and meetings). In the course of time, we usually install a lot of toolsets, the configurations that have evolved over a long period of time. 
But what if we have to set up a new PC, should I go over all that config evolution again ? well optimally speaking obviously not. 
What if I occasionally login to a server and manage some scripts, I would like to have the server behave as close as my local machine with the same config. So, should I first set up the server  first? Well, optimally speaking obviously not. 

I would like to keep track of changes, and commits over time. 


Here are the dotfiles I have been using. 


Lets focus on some important configs such as neovim (nvim ), alacritty, polybar, i3 etc. 
Each of them has multiple directories and files. 


## Stow 
Stow or GNU Stow is simply a symlink manager. In simple terms, it can be considered as creating shortcuts. It operates on directories, and files.

Installations 
```apt install stow``` 
```brew install stow```

( btw i don't use arch ) 

Usage :
Suppose I have directory structure as : ``` folder1/folder2/folder3/file.py``` 


Now, let's say I am in folder2 which is the stow directory from where I will be running stow. 
In that case, it will attempt to stow folder3. folder3 is the directory that is going to stow. 
And folder1 is the place where it will stow folder3 i.e folder1 would be the target directory. 

Target directory is always the parent directory, the thing that is going to stow is the child directory from the directory where stow is executed(stow directory) if using default.












Running `ls -la``  on target directory(folder1) gives the following output before executing stow.
 

We have no symlinks created as of now. 

Now lets move to folder 2 ```cd folder2``, and run ```stow .``` 



We do not see any logs. That is expected.

Now, let's go back to folder1 which is our target directory, and check if there have been any changes. 

``` cd .. && ls -la``` 



Well, now we see we have a new directory which is folder3 which is the symlink of the source directory. It's not copying, it's just pointing to the folder. The changes that you make in any source or target directory will be reflected in both of them. 

Tips: 
To remove the symlink just run ``` stow -D .``` from folder1. 
### Managing dotfile 

We have dotfiles in $HOME/.config/ as in fig …

Lets create a folder named my_prettty_awesome_dotfiles in $HOME which will store all of our dotfiles, and would also be a pretty git repository. 
```mkdir ~/my_prettty_awesome_dotfiles``` 

For demonstration, let's say I want to manage my alacritty config which is alacritty.yml 

I will create a folder named alacritty in my_prettty_awesome_dotfiles which will contain my alacritty configs. 

```mkdir ~/my_prettty_awesome_dotfiles/alacritty```

To easily symlink between our config which is in $HOME/.config and the source directory. We would create a directory structure as : 

```my_pretty_awesome_dotfiles/alacritty/.config/alacritty/alacritty.yml``` 

Now, from the ``my_pretty_awesome_dotfiles``` I would stow alacritty. 

```stow alacritty``` 

Before that, lets verify that we do not have alacritty folder in $HOME/.config



It does not exist. 

Now, execute, ``stow alacritty``` from ```$HOME/my_pretty_awesome_dotfiles```, and check if any changes in ```$HOME/.config``` 


A symlink alacritty is created which points to ```$HOME/my_pretty_awesome_dotfiles/alacritty/.config/alacritty``` which is what we needed. 

If we go to the alacritty symlink we would find the alacritty.yml which is my alacritty config. I got my alacritty config ready. Yehhhhh !!  



Now, I would do the same for each of my tools. Also, I would initialize the my_pretty_awesome_dotfiles as a git repository and track the history. If unfortunately, if you break something, just pull the previous working commit and stow that. You are good to go. 



References:
GNU stow  : https://www.gnu.org/software/stow/manual/stow.html 
My dotfiles : https://github.com/prajinkhadka/dotfiles__ 

