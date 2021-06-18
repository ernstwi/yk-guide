# Yubikey setup/GPG - new keys
## Mac
### Prerequisites
* YubiKey 5 Series
* brew
* a usb-memory

### Installation 
Make sure that `/opt/homebrew/bin` (if using an ARM Mac) or `/usr/local/bin` (x86) is before `/bin` in your `$PATH`. This should be the case with a standard Homebrew installation.

1. `$ brew install ykman`
2. `$ brew install pinentry-mac`
3. `$ brew install gnupg`

This guide assumes that you are using Bash. Note that this is not the default shell on macOS Catalina and forward.

You can install an up-to-date version of Bash using Homebrew:

1. `$ brew install bash`
2. `$ sudo bash -c "echo $(which bash) >> /etc/shells"`
3. `$ chsh -s $(which bash)`

### Create a more secure workspace
1. `$ umask 077`
2. `$ diskutil erasevolume HFS+ 'RAMDisk' $(hdiutil attach -nomount ram://8388608)`
3. `$ cd /Volumes/RAMDisk`
4. `$ mkdir key_space`
5. `$ cd key_space`
6. `$ chown -R $(whoami) .`
7. `$ chmod 700 .`

### Update gpg.conf
Create a gpg.conf in `key_space` containing:
```
default-preference-list SHA512 SHA384 SHA256 SHA224 AES256 AES192 AES CAST5 ZLIB BZIP2 ZIP Uncompressed
personal-digest-preferences SHA512
cert-digest-algo SHA512
```

### Create primary Key
`$ export GNUPGHOME=$PWD`

Kill any running `gpg-agent` process.

`$ gpg --full-gen-key`

1. `(1) RSA and RSA (default)`
2. 4096
3. 1y
4. Add `Real name` and `Email` but leave `Comment` empty

Keep the returning key ID handy.

### Edit primary key
`$ gpg --edit-key --expert <ID>`

#### Add another UID (Optional)
`gpg> adduuid` # until done

#### Add Authenticate subkey (SSH)
`gpg> addkey`
1. `(8) RSA (set your own capabilities)`
2. toggle S & E & A which should result in only "Authenticate".
3. 4096
4. 1y

#### List keys
```
gpg> list
sec  rsa4096/0487C6AE38A74C12
     created: 2021-05-28  expires: 2022-05-28  usage: SC
     trust: ultimate      validity: ultimate
ssb  rsa4096/2E180F6B22DFF0D0
     created: 2021-05-28  expires: 2022-05-28  usage: E
ssb  rsa4096/E641CAE29209F416
     created: 2021-05-28  expires: 2022-05-28  usage: A
[ultimate] (1). Magnus Svensson <masv@sunet.se>
```

`gpg> save`

### Backup
When writing the keys to your card, gpg2 will delete the existing keys on your keyring and replace them with a stub. If you want a backup of the keys to store offline in a safe place, then now is the time to make a backup:

`$ gpg --export-secret-key --armor > /Volumes/<USB_NAME>/secretkey.backup`

`$ gpg --output /Volumes/<USB_NAME>/revoke.asc.backup --gen-revoke <ID>`


### Export public key
You will also need to save your public key somewhere, so you can import it in your real GPG keyring. Something like
`$ gpg --armor --export <ID> > ~/my-public-key.asc`

### Card modifications
#### Change Admin PIN/PIN/Reset Code
`$ gpg --edit-card`

1. `admin`
2. `passwd`

Follow the instructions...

#### Set card holder nam
`$ gpg --edit-card`

1. `name`

Follow the instructions...

### Write to card
Place each key (capability) on card. This is a general instruction.

`$ gpg --edit-key <ID>`

`gpg> keytocard` # primary key to card (Sign)

`gpg> key 1`

`gpg> keytocard`

`gpg> key 1`

`gpg> key 2`

`gpg> keytocard`

`gpg> key 2`

`gpg> save`


### Import public key into the real keychain
`$ unset GNUPGHOME`

Kill `gpg-agent`.

`$ gpg --import ~/my-public-key.asc`

`$ gpg --edit-key <ID>`

`gpg> trust`

set 5

`gpg> save`

### Verify keys on card
```
$ gpg -K
/Users/masv/.gnupg/pubring.kbx
------------------------------
sec>  rsa4096 2021-05-28 [SC] [expires: 2022-05-28]
      D0B2D9FF86A6B3A6314E632C5BA4BA8D2A3E5C61
      Card serial no. = 0006 10124562
uid           [ultimate] Magnus Svensson <masv@sunet.se>
ssb>  rsa4096 2021-05-28 [E] [expires: 2022-05-28]
ssb>  rsa4096 2021-05-28 [A] [expires: 2022-05-28]
```

### Enable SSH support
`$ echo "enable-ssh-support" >> ~/.gnupg/gpg-agent.conf`

`$ echo "pinentry-program $(which pinentry-mac)" >> ~/.gnupg/gpg-agent.conf`

`$ echo "export SSH_AUTH_SOCK=$(gpgconf --list-dirs agent-ssh-socket)" >> ~/.bash_profile`

`$ echo "gpgconf --launch gpg-agent" >> ~/.bash_profile`

`$ gpg --export-ssh-key <ID> > ~/.ssh/id_rsa_yubikey.pub`

Logout/login (Is this the only way of doing it?)

`ssh-add -L` lists ssh public-keys


### Clean up RAMDisk
`$ rm -Pr private-keys-v1.d/`

`$ rm -P pubring.kbx*`

`$ cd .. && rm -rf key_space/`

`$ sudo diskutil unmount force RAMDisk/`


#### Enable Touch policies (Encouraged)
A touch policy takes advantage of the fact that the Yubikey is a HSM that can be restricted by touch activation.

`$ ykman openpgp keys set-touch <AUT|SIG|ENC> ON`


### Test sign
`$ gpg --output /tmp/test_sign.sig --sign /tmp/test_sign`

`$ gpg --verify /tmp/test_sign.sig`

Compare the returning key fingerprint with the sub key fingerprint with Sign capabilities, it should be equal.

`$ gpg --list-keys --with-subkey-fingerprints`


### Test encrypt
`$ gpg -e -r <YOUR_EMAIL> /tmp/test_encrypt.txt`

`$ gpg --decrypt /tmp/test_encrypt.txt.gpg`

should return: 

```
gpg: encrypted with rsa4096 key, ID <ENCRYPT_SUBKEY_ID>, created <DATE>
      "<NAME> <<EMAIL>>"
```