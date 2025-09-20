# About
- procedure to change cert on TrueNAS
- using smallstep to provide the new (self-signed) certificate

# TrueNAS: Generate CSR
- Menu: `Credentials` -> `Certificates`
	- card: `Certificate Signing Requests` -> `Add`
		- 1 - Identifier and Type
			- **Name**: <a descriptive name>
			- Type: CSR
			- Profile: RSA
		- 2 - Certificate options
			- Type: RSA
			- Key Lenght: 4096
			- Digest Algorithm: SHA256
		- 3 - Certificate subject
			- Country: <your data>
			- State: <your data>
			- Locality: <your data>
			- Organization: <your data>
			- Email: <your data>
			- **Common Name**: `<your FQDN>`
			- **Subject Alternate Names**: `<your FQDN>`
				- may also add IP here
		- 4 - Extra constraints
			- leave defaults
				- may e.g. remove `CLIENT_AUTH`
		- 5 - Confirm options
			- Save
- get text of generated `.csr` file
	- use either download button or edit (and copy text from there)
# Sign Certificate with `step-ca`
- log in to the smallstep docker container
	- e.g. `docker exec -it step-ca /bin/bash`
	- alternatively, any machine with `step-cli` installed
- create a file `truenas.csr` and paste the CSR there
- sign CSR with the intermediate key
	- ```bash
		step certificate sign truenas.csr certs/intermediate_ca.crt secrets/intermediate_ca_key --not-after=2026-01-05T00:00:00Z
		```
		- see [smallstep docs - step certificate sign](https://smallstep.com/docs/step-cli/reference/certificate/sign/)
		- note the time should be in RFC 3339, or use duration with valid time units mentioned in docs
- enter intermediate key password
	- this is the password under `/home/step/secrets/password` inside the container
	- copy certificate text from output
# Import and Apply the Full Chain in TrueNAS
- Menu: `Credentials` -> `Certificates`
- card: **Certificates** -> **Add**
	- 1 - Identifier and Type
		- Name: <a descriptive name>
		- **Type**: `Import Certificate`
	- 2 - Import certificate
		- 💡❗ **concatenate both**
			- server cert from CA from previous step (pun intended 😉)
			- intermediate cert
				- in `/docker/step-ca/data/certs/intermediate_ca.crt`
		- ❗this is important, as otherwise the chain is broken and you get problems in browsers!
		- chk: **CSR exists on this system** -> ON
			- -> select the previously created CSR
	- 3 - Confirm options
		- confirm and **save**
# Use the new cert
- Menu: `System Settings` -> `General`
- card: **GUI** -> btn: **Settings**
	- **GUI SSL Certificate**: Select new cert
	- -> **Save**
	- confirm restarting web service
- re-login
# Clean up old cert
- menu: Credentials -> Certificates
- delete old certs
	- this is not possible with the currently used cert
	- **Force** chk should **not** be needed
