---
title: "Lecture Notes: Getting Started"
date: 2020-01-02 01:01:00
categories: notes lecture 
layout: post
---

This set of notes will guide you through the initial steps to get up and running in this course.
### First Steps

To make the most out of the first few lectures, it’s a good idea to complete the following tasks as soon as possible:

1. **Watch the [Tips and Tricks videos](https://youtube.com/playlist?list=PLeKxIn6N-kCi38WxOqNBXhxrZnOE9SVET)**: These videos feature advice from former students on what they think you should know before diving into this course.
    
2. **Read the [course syllabus](https://cs4401.walls.ninja/syllabus)**: Familiarize yourself with the course structure, policies, and expectations.
    
3. **Create an account on the [course infrastructure](https://cs4401.walls.ninja/)**: If you’re reading this, you’ve likely already completed this step.
    
4. **Join the course Discord server**: Use the link provided to you. From now on, we’ll be using Discord for all course communications instead of email. Please set your Discord nickname to your real name.
    
5. **Check the [schedule](https://cs4401.walls.ninja/schedule)**: Take a look at the schedule for the first few lectures to know what’s coming up.
    
6. **Set up your local environment for challenge solving**: Follow the instructions below to ensure you’re ready to tackle the challenges.
    

**Optional Preparation:**If you’d like a refresher or an introduction to using command-line interfaces, I highly recommend working through the [Bandit challenges](https://overthewire.org/wargames/bandit/).

### Working with the Course Server

I’ve set up a server environment specifically for this course, where you can solve most of the challenges. This server comes preloaded with essential tools like `vim`, `tmux`, `pwntools`, and `gdb`, but there are still some configurations you’ll need to set up based on your preferences.

Here’s a quick overview of these tools:

- **Vim**: A powerful text editor for writing and editing code directly from the command line.
- **Tmux**: A terminal multiplexer that allows you to run multiple terminal sessions within a single window, making it easier to manage your workflow.
- **Pwntools**: A CTF (Capture the Flag) framework and exploit development library, perfect for writing exploits in Python.
- **GDB (GNU Debugger)**: A powerful debugger that lets you inspect and control the execution of programs, essential for understanding how binaries work and for developing exploits.

During lectures, I’ll be using this server to solve challenges, and I recommend you use it as well, especially if you have a computer with Apple Silicon. Apple Silicon uses a different hardware architecture than the challenge binaries, which can complicate the use of tools like GDB, even when using Docker. If you have an x86-based machine, the Docker-container method mentioned later in these notes is also a great option.

To get up and running with the server, you’ll need to:

1. Set up your SSH keys for secure access.
2. Install GEF (GDB Enhanced Features) to make debugging easier.
3. Configure `tmux`.
4. Optionally, configure vim.

### Setting up SSH

We’ll primarily use SSH to connect to the course server. **SSH (Secure Shell)** is a protocol that allows you to securely connect to another computer over a network. It’s like opening a terminal on your computer, but instead, you’re controlling a remote machine.

To connect to the course server, open up a terminal on your personal machine and run the following command:

```
ssh username@cs4401shell.walls.ninja
```

Let’s break down this command:

- **`ssh`**: This tells your computer that you want to start a secure shell connection.
- **`username`**: Replace this with your WPI username.
- **`cs4401shell.walls.ninja`**: This is the address of the course server you’re connecting to.


If everything goes smoothly, you’ll be prompted to enter your platform password (the same one you use for the course website). After entering your password, you should see a command prompt on the course shell server, where you can type and run commands just like you would on your own machine.

**Pro tip:** Make sure to include "shell" in the URL "cs4401shell.walls.ninja." If you forget and just type "cs4401.walls.ninja," you’ll end up trying to connect to the course's web server instead of the shell server. You won’t need to access the course web server directly.

**A caveat for Windows users:** If you’re on Windows, you’ll need to install Windows Subsystem for Linux (WSL) or use a program like [PuTTY](https://www.putty.org/) to access SSH. I recommend WSL for a smoother experience. If you’re on a Mac or Linux machine, you can use the command directly in your terminal.

#### Connecting with SSH keys

Connecting via SSH requires authentication each time. While you can manually type your password, that can get tedious. Instead, you can set up SSH keys for a smoother experience.

**What are SSH keys?** SSH keys are a pair of cryptographic keys—a public key and a private key—that are used to securely connect to a server without needing to enter a password each time. The public key is stored on the server, while the private key remains on your local machine.

To set up SSH keys, you’ll use the `ssh-keygen` and `ssh-copy-id` commands. The specifics might vary slightly depending on your operating system.

First, generate the SSH key pair:

   ```bash
   ssh-keygen -C "username@wpi.edu"
   ```

Here `-C "your_email@example.com"` adds a comment to the key, specifically your email.

When prompted to "Enter a file in which to save the key," press `Enter` to accept the default location (e.g., `~/.ssh/id_ed25519`), or specify a different path if you prefer. The default path is typically fine.

You’ll also be prompted to enter a passphrase. This is optional—you can enter a passphrase for extra security or leave it empty for convenience.

Your keys will be saved in the following files. The specific names depend on the type of key you generated and the path you specified previously. 

- **Public key**: `~/.ssh/id_ed25519.pub`
- **Private key**: `~/.ssh/id_ed25519`

As the names suggest, your public key can be shared with others (in this case, the course server), but you should keep your private key secure and private.

Next, copy your public key to the course server. The easiest way to do this is with `ssh-copy-id`:

   ```bash
   ssh-copy-id -i ~/.ssh/id_ed25519.pub username@cs4401shell.walls.ninja
   ```
   
Replace `username` with your WPI username. You’ll be prompted to enter your platform password. After that, your public key will be added to the `~/.ssh/authorized_keys` file on the course server.

If `ssh-copy-id` isn’t available, you can manually copy the public key:

1. Print out the contents of your public key file:

   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```

Copy the entire string. Then ssh into the course server. Create the `.ssh` directory on the remote machine, if it doesn't exist, and set the permissions:

   ```bash
   mkdir ~/.ssh
   chmod 700 ~/.ssh
   ```

Finally, you'll Add your public key to the `authorized_keys` file and set the permissions:

   ```bash
   echo "your_copied_public_key" >> ~/.ssh/authorized_keys
   chmod 600 ~/.ssh/authorized_keys
   ```

Now you should be able to SSH into the course server without being prompted for a password:

```bash
ssh -i ~/.ssh/id_ed25519 username@cs4401shell.walls.ninja
```

The `-i ~/.ssh/id_ed25519` option specifies which private key to use, but this may not be necessary depending on your setup.
#### Making Life Even Easier with an Alias

To make connecting even simpler, you can create an alias in your shell configuration file (e.g., `~/.zshrc` for `zsh` users):

```bash
alias cs4401shell="ssh -i ~/.ssh/id_ed25519 username@cs4401shell.walls.ninja"
```

With this alias, you can connect to the server by simply typing `cs4401shell`. If you’re using a different shell (like `bash`), you’ll need to add the alias to your specific shell’s configuration file.

### Final Configuration Steps

I’ve already set up the course shell server with most of the tools you’ll need to solve the challenges. However, there are a few additional steps you’ll need to take to complete your environment.

#### Installing GEF for GDB

First, you’ll need to install GEF, which is a plugin for GDB that makes debugging much easier and more pleasant to use. GEF adds a bunch of useful features like better visualization of data structures and easier navigation through the code. To set up GEF, run the following commands on the course server:

```bash
cd && \
git clone https://github.com/hugsy/gef.git
cd gef
git checkout 2024.06
cp gef.py ~/.gdbinit-gef.py
echo source ~/.gdbinit-gef.py >> ~/.gdbinit
```

This command downloads the GEF plugin and configures GDB to load it automatically whenever you start a debugging session.
#### Setting Up Tmux

Next, we’ll configure Tmux to automatically start whenever you connect to the shell server. Tmux is a terminal multiplexer that allows you to manage multiple terminal sessions within a single window. By setting this up, you gain some important features like the ability to resume shell sessions automatically and the convenience of launching a debug session directly from your exploit scripts.

To do this, we’ll modify the `.bash_aliases` file to ensure that Tmux launches automatically:

```bash
export TZ="America/New_York"

# Automatically start tmux if not already running
# Try to attach to session "default", creating it if not available.
if [ -z "$TMUX" ]; then
    tmux attach-session -t default || tmux new-session -s default
fi
```


Now, let’s download a configuration file for Tmux. This step is optional, but I recommend using this configuration so that your shortcuts and Tmux behavior match what I’ll be demonstrating in class:

```bash
cd && \
curl -O https://raw.githubusercontent.com/rjwalls/EpicTreasure/master/24.04/tmux.conf && \
mv tmux.conf .tmux.conf
```

Feel free to customize this file if you want to personalize your setup.

**Note on scrolling in Tmux**: Scrolling can be a bit tricky in Tmux. If you’re using the configuration file above, you can scroll by pressing `Ctrl-a` followed by `[` and then using the arrow keys. Press `Esc` to stop scrolling and return to input mode.

#### Configuring Vim

Finally, if you prefer to use Vim as your text editor, here’s how you can match my basic setup.

**Step 1: Install Pathogen.** Pathogen is a Vim plugin manager that makes it easy to install and manage Vim plugins. Run the following command to set it up:

```bash
mkdir -p ~/.vim/autoload ~/.vim/bundle && curl -LSso ~/.vim/autoload/pathogen.vim https://tpo.pe/pathogen.vim
```

**Step 2: Install a Color Scheme**. Next, let’s install a color scheme for Vim that’s easy on the eyes:

```bash
cd .vim/bundle/ && \
git clone https://github.com/nielsmadan/harlequin.git
```

**Step 3: Update Your `.vimrc`**

Now, update your `.vimrc` file with the following content to enable Pathogen and apply some useful settings:

```vim
execute pathogen#infect()
syntax on
filetype plugin indent on

colorscheme harlequin


" Map semi-colon to colon
nmap ; :
" Map to allow double press of semi-colon to take over the former duties of semi-colon
noremap ;; ;

" Map jj to escape
imap jj <Esc>

" Use leader+r to run the current open file
augroup RunCurrentFile
    autocmd!
    autocmd FileType python nnoremap <leader>r :!python3 %<CR>
    autocmd FileType sh nnoremap <leader>r :!bash %<CR>
    autocmd FileType c nnoremap <leader>r :!gcc % -o %< && ./%<<CR>
augroup END




```

This setup maps some common shortcuts to make coding in Vim more efficient and includes a few handy commands for running your code directly from the editor.

**Step 4: Install the vim-sensible Plugin**. Finally, install the [vim-sensible](https://github.com/tpope/vim-sensible) plugin, which provides a set of reasonable defaults for Vim:

```bash
cd .vim/bundle/ && \
git clone https://github.com/tpope/vim-sensible.git
```

With these configurations, you should have a well-rounded setup that matches what I’ll be using during lectures.

### Solving Locally with the Course Docker Container


Sometimes, you might prefer or need to work directly on your local machine—especially if you have an unreliable internet connection. In these situations, I recommend using the [Epic Treasure](https://github.com/rjwalls/EpicTreasure) Docker container. This container is preloaded with a variety of useful tools that will help you solve most of the course challenges. **Note: You should only use this container if you have an x86-based computer. Tools in this container, e.g., GDB, will not work as expected on Apple Silicon. 

**Step 1: Install Docker**. Before you can use the Epic Treasure container, you’ll need to install Docker on your machine. Docker is a platform that lets you run applications in lightweight, isolated environments called containers.

**Step 2: Launch the Container**. Once Docker is installed, you can launch the container by running the following command in your shell:

```
docker run --privileged -v ${PWD}:/root/host-share --rm -it --workdir=/root rjwalls/epictreasure tmux
```

Let’s break down what this command does:

- **`--privileged`**: Grants the container access to all devices on the host machine.
- **`-v ${PWD}:/root/host-share`**: Mounts your current directory (where you run the command) to a directory inside the container, allowing you to share files between your machine and the container.
- **`--rm`**: Automatically removes the container when it exits, so you don’t have to clean up afterward.
- **`-it`**: Runs the container in interactive mode, so you can interact with it through your terminal.
- **`--workdir=/root`**: Sets the working directory inside the container to `/root`.
- **`rjwalls/epictreasure`**: The name of the Docker image you’re using.
- **`tmux`**: Launches a Tmux session inside the container.

**Note for Windows Users**: Some Windows users might experience issues with Tmux, such as text printing incorrectly (e.g., scrolling repeated text). If you encounter these problems or prefer not to use Tmux, you can simply remove `tmux` from the command:

```bash
docker run --privileged -v ${PWD}:/root/host-share --rm -it --workdir=/root rjwalls/epictreasure`
```


**Step 3: Disable ASLR in the Container**. To disable Address Space Layout Randomization (ASLR) in the Docker container, run the following command after launching the container:

```bash
echo 0 | sudo tee /proc/sys/kernel/randomize_va_space
```

**Important:** You must run this command each time you launch a new Epic Treasure container.
