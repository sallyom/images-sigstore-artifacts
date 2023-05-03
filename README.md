# Podman Sigstore Image Trust

## Non-root `podman` configuration

The files for root podman are in `/etc/containers`. You should use non-root podman whenever possible.
To configure podman image trust with rootful podman, replace `~/.config` with `/etc` below.

### Configure `podman` to read & write sigstore signatures 

```sh
mkdir ~/.local/share/containers/sigstore
mkdir -p ~/.config/containers/registries.d
sudo cp -R /etc/containers/registries.d ~/.config/containers/
sudo chown -R somalley:somalley ~/.config/containers

ls -al ~/.config/containers/registries.d/
total 16
drwxr-xr-x. 1 somalley somalley  166 May  3 12:21 .
drwxr-xr-x. 1 somalley somalley  148 May  3 12:34 ..
-rw-r--r--. 1 somalley somalley 1137 May  3 12:21 default.yaml
-rw-r--r--. 1 somalley somalley  120 May  3 12:21 registry.access.redhat.com.yaml
-rw-r--r--. 1 somalley somalley   99 May  3 12:21 registry.redhat.io.yaml

```

Edit `$HOME/.config/containers/registries.d/quay.yaml` to configure your personal container image repository

```yaml
docker:
  quay.io/sallyom:
    sigstore: file:///home/somalley/.local/share/containers/sigstore
    sigstore-staging: file:///home/somalley/.local/share/containers/sigstore
    use-sigstore-attachments: true
```

### Configure podman image trust

The following policy will only pull & run images from `quay.io/sallyom` if they
are both signed by Sigstore private key and also signed with GPG & somalley@redhat.com identity.
Images from redhat certified container registry are also allowed only if signed with GPG.

| :memo:        | Export and save your gpg key to a local file        |
|---------------|:----------------------------------------------------|


Edit `~/.config/containers/policy.json`

```json
{
    "default": [
        {
            "type": "reject"
        }
    ],
    "transports": {
        "docker": {
	    "registry.access.redhat.com": [
		{
		    "type": "signedBy",
		    "keyType": "GPGKeys",
		    "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
		}
	    ],
	    "registry.redhat.io": [
		{
		    "type": "signedBy",
		    "keyType": "GPGKeys",
		    "keyPath": "/etc/pki/rpm-gpg/RPM-GPG-KEY-redhat-release"
		}
	    ],
        "quay.io/sallyom": [
                {
                    "type":    "sigstoreSigned",
                    "keyPath": "/home/somalley/image-signing-keys/cosign.pub",
                    "signedIdentity": {"type": "matchRepository"}
                },
		{
		    "type": "signedBy",
		    "keyType": "GPGKeys",
		    "keyPath": "/home/somalley/image-signing-keys/somalley.gpg"
		}
        ]
	},
        "docker-daemon": {
	    "": [
		{
		    "type": "insecureAcceptAnything"
		}
	    ]
	}
    }
}

```

### Podman Image Trust

```bash
podman image trust show
TRANSPORT      NAME                        TYPE            ID                   STORE
all            default                     reject                               file:///home/somalley/.local/share/containers/sigstore
repository     quay.io/sallyom             sigstoreSigned  N/A                  file:///home/somalley/.local/share/containers/sigstore
repository     quay.io/sallyom             signed          somalley@redhat.com  file:///home/somalley/.local/share/containers/sigstore
repository     registry.access.redhat.com  signed          security@redhat.com  https://access.redhat.com/webassets/docker/content/sigstore
repository     registry.redhat.io          signed          security@redhat.com  https://registry.redhat.io/containers/sigstore
docker-daemon                              accept                               file:///home/somalley/.local/share/containers/sigstore
```

### Test your podman trust policy

| :memo:        | A signed and unsigned image are available in your container image registry        |
|---------------|:----------------------------------------------------------------------------------|

```bash
podman image pull quay.io/sallyom/simplewebserver:unsigned
Trying to pull quay.io/sallyom/simplewebserver:unsigned...
Error: Source image rejected: A signature was required, but no signature exists

podman image pull quay.io/sallyom/simplewebserver:signed
Trying to pull quay.io/sallyom/simplewebserver:signed...
Getting image source signatures
Checking if image destination supports signatures
Copying blob 78aee8e6e64c skipped: already exists  
Copying blob 867858e30946 skipped: already exists  
Copying blob 4ee7981689b9 skipped: already exists  
Copying blob d94b05d5bf44 skipped: already exists  
Copying config 256ae4d00e done  
Writing manifest to image destination
Storing signatures
256ae4d00e3f9714a75af239fcd804293b0fd8ac59fb9fa85d34ab54fef52a5e

```

### How to sign images with podman

You should always sign and push images as part of an automated CI/CD process.
Here is an example of how to sign with podman.

```sh
podman build -t quay.io/sallyom/simplewebserver:unsigned .
podman push --sign-by-sigstore-private-key ~/image-signing-keys/cosign.private --sign-by somalley@redhat.com quay.io/sallyom/simplewebserver:signed
```