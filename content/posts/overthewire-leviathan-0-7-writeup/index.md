+++
title = 'OverTheWire — Leviathan 0 – 7 Writeup'
date = '2023-02-15'
draft = false
tags = ["Overthewire"]
summary = '“Dare you face the lord of the oceans?”'
[cover]
image = 'main.png'
+++

# Level 0 → 1

To get into the first level, SSH as `leviathan0` into `leviathan.labs.overthewire.org`.

```bash
ssh leviathan0@leviathan.labs.overthewire.org -p 2223
```
This level is pretty simple, we can list out the only file that stands out; `.backup/bookmarks.html`, piping the output to `grep` we can get the password to **leviathan1**.

{{< figure src="image_1.png" align="center" >}}

# Level 1 → 2

We have an executable called `check`. Once we try to run it, it prompts for a password.

Running it with `ltrace` shows us what our input is being compared to, in this case `sex`.

Now all we have to do is use the correct password and cat the `/etc/leviathan_pass/leviathan2` file for the next level’s password.

{{< figure src="image_2.png" align="center" >}}

# Level 2 → 3

This one gets a bit more complicated.

We have an executable… again, obviously, using it to print the next level’s password doesn’t work.

Create a file in the `/tmp` directory that injects a command into the executable once it’s run.

```bash
touch "/tmp/"file;bash"
```

Execute printfile, passing the file we created earlier.

```bash
./printfile /tmp/file\;bash
```

The semicolon followed by `bash` in our filename spawns a shell, and we get **leviathan3’s privileges** because the [SUID bit](https://www.redhat.com/sysadmin/suid-sgid-sticky-bit) is set on `printfile`, so once bash executes, it will be executed as **leviathan3**.

{{< figure src="image_3.png" align="center" >}}

# Level 3 → 4

Yet again, another executable.

If we examine `strcmp()` via `ltrace`, we can see that the correct password for the executable is `snlprintf`.

{{< figure src="image_4.png" align="center" >}}

# Level 4 → 5

Examining the only interesting directory; `.trash`, we find the file `bin` which prints binary output.

Google’s your friend here — convert the binary output to ASCII to get the next level’s password.

{{< figure src="image_5.png" align="center" >}}

# Level 5 → 6

[Symbolic links](https://en.wikipedia.org/wiki/Symbolic_link).

The executable `leviathan5` tries to read the file `/tmp/file.log` but can’t find it.

Let’s help it out — create a symbolic link to `/etc/leviathan_pass/leviathan6` and run the executable.

{{< figure src="image_6.png" align="center" >}}

# Level 6 → 7

We need the correct 4 digit code to run this executable, time to brute force.

{{< figure src="image_7.png" align="center" >}}

Let’s make a temporary directory to house our brute force script.

```bash
mktemp -d
```

{{< figure src="image_8.png" align="center" >}}

Open a file using nano or vim, call it whatever you want. Now for the important part, brute forcing the `leviathan6` script.

{{< figure src="image_9.png" align="center" >}}

Give your brute force script a bit to run, and a little later, we’ll have brute forced the 4 digit code.

{{< figure src="image_10.png" align="center" >}}

# Level 7

{{< figure src="image_11.png" align="center" >}}