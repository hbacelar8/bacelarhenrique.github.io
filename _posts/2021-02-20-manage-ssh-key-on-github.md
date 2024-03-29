---
title:  "Add and Configure SSH Keys on Github"
excerpt: How to set up a SSH key to your Github account and configure it locally.
tags: git tools
last_modified_at: "2021-02-27"
header:
  teaser: assets/images/git.jpeg
  overlay_image: assets/images/git.jpeg
  overlay_filter: 0.5
---

# Preface
In this article I'll explain how to set up a SSH key, add it to you Github account and configure it locally.

# Generating a new SSH key
Run the following command on a terminal to create a new private and public SSH key pair, replacing with your personal email account

```bash
ssh-keygen -t rsa -C "personal@mail.com" -f ~/.ssh/id_rsa_github
```

With this, two different files will be created on the folder ```~/.ssh```

```
~/.ssh/id_rsa_github
~/.ssh/id_rsa_github.pub
```

# Adding the key to Github
Open the ```id_rsa_github.pub``` file and copy all its content.

Head to your Github account. At the top right corner, click on your profile picture then **Settings**. Click on **SSH and GPG keys** on the left side menu and then ```New SSH key```.

<figure>
    <a href="/assets/images/ssh-key.png"><img src="/assets/images/ssh-key.png"></a>
</figure>

Paste the public key that has been copied at the Key box, choose a title (i.e. windows-desktop) and click on the ```Add SSH key``` button.

# Registering a SSH key
Start a SSH agent with

```bash
eval `ssh-agent -s`
```

Register the key by running

```bash
ssh-add ~/.ssh/id_rsa_github
```

# Editing a ```config``` file
Create a configuration file that will add the different SSH keys you have for all git providers and emails that you will eventually create.

```
touch ~/.ssh/config     // Create config file if it does not exist
code ~/.ssh/config .    // Edit config file
```

The ```config``` file should look like this

```
# This is a comment

# Personal github account
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa_github
```

# Testing your SSH connection
After you've set up your SSH key and added it to your GitHub account, you can test your connection with

```bash
ssh -T git@github.com
```
You should see a warning like this

```
The authenticity of host 'github.com (IP ADDRESS)' can't be established.
RSA key fingerprint is SHA256:nThbg6kXUpJWGl7E1IGOCspRomTxdCARLviKw6E5SY8.
Are you sure you want to continue connecting (yes/no)?
```

Type ```yes``` and you should finally see the message

```
Hi username! You've successfully authenticated, but GitHub does not provide shell access.
```

Verify that the resulting message contains your username. If you receive a "permission denied" message, see [Error: Permission denied (publickey)](https://docs.github.com/en/github/authenticating-to-github/error-permission-denied-publickey){:target="_blank"}.

After following the above steps, you should be able clone and edit your GitHub repositories on your local machine.

# Afterword
The same process can be used to create multiple SSH keys and add them to different git providers, making sure to add new entries on the ```config``` file.