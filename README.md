# openssl-cipher-list-scan



A rework of the script found on the [ISE](https://www.ise.io/blog/archives/using-openssl-determine-ciphers-enabled-server/index.html) page. The 'openssl-cipher-list-scan' script can show you any HTTP/ FTP server:

  - Certificate Subject
  - Certificate Issuer
  - Supported Server Cipher(s)

# Installation:

Download the 'cipherlist-host_port_protocol.sh' file from the releases page and change the file permissions. The second command is only needed if you receive a bad interpreter error.
 ```sh
$ chmod 700 ./cipherlist-host_port_protocol.sh
$ sed -i 's/\r//' ./cipherlist-host_port_protocol.sh
```

# Usage:

 ```sh
$ ./cipherlist-host_port_protocol.sh 192.168.1.1 443 http
$ ./cipherlist-host_port_protocol.sh 192.168.1.1 21 ftp
```

# Script Content:

 ```sh
#!/bin/sh

if [ $# -ne 3 ]; then  
cat << END	
There is missing information, please provide: <IP> <PORT> <http/ftp>
END
exit 1
fi

case "$3" in
	ftp)

if nc $1 $2 < /dev/null > /dev/null 2>&1; then
	echo "Starting OpenSSL TLS scanning for $1:$2"
  else
	echo "No connection could be established with $1:$2 (check the provided IP and PORT)"
  exit 1
fi

e=$(for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -starttls ftp -cipher $c -no_tls1_3 < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done
	for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -starttls ftp -cipher $c < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done)
 
d=$(echo $e | rev | cut -d ' ' -f 1 | rev)

if [ -z "$d" ]; then
  echo -e "\nTLS connection to $1:$2 has failed (make sure that FTPS is enabled)"
else 

checkDSA=`openssl s_client -connect $1:$2 -starttls ftp -cipher $d -no_tls1_3 < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'`

checkTLS13=`openssl s_client -connect $1:$2 -starttls ftp -cipher $d < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'`

[ -z "$checkDSA" ] && echo -e "\n$checkTLS13\n" || echo -e "\n$checkDSA\n"

  echo -e 'Supported Server Cipher(s):'
		for v in ssl3 tls1 tls1_1 tls1_2; do
			for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -starttls ftp -cipher $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls+/TLSv/g' | tr '_' '.' 
		done
		done
		for v in tls1_3; do
			for c in $(openssl ciphers 'tls1:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -starttls ftp -ciphersuites $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls1_3+/TLSv1.3/g' | tr '_' '-'  
		done 
		done
  exit 1
fi
;;
	http)
		
if nc $1 $2 < /dev/null > /dev/null 2>&1; then
	echo "Starting OpenSSL TLS scanning for $1:$2"
  else
	echo "No connection could be established with $1:$2 (check the provided IP and PORT)"
  exit 1
fi

e=$(for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -cipher $c -no_tls1_3 < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done
	for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -cipher $c < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done)
 
d=$(echo $e | rev | cut -d ' ' -f 1 | rev)

if [ -z "$d" ]; then
  echo -e "\nTLS connection to $1:$2 has failed (make sure that HTTPS is enabled)"
else 

checkDSA=`openssl s_client -connect $1:$2 -cipher $d -no_tls1_3 < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'`

checkTLS13=`openssl s_client -connect $1:$2 -cipher $d < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'`

[ -z "$checkDSA" ] && echo -e "\n$checkTLS13\n" || echo -e "\n$checkDSA\n"

  echo -e 'Supported Server Cipher(s):'
		for v in ssl3 tls1 tls1_1 tls1_2; do
			for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -cipher $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls+/TLSv/g' | tr '_' '.' 
		done
		done
		for v in tls1_3; do
			for c in $(openssl ciphers 'tls1:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -ciphersuites $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls1_3+/TLSv1.3/g' | tr '_' '-'  
		done 
		done
  exit 1
fi
;;	

	* )
echo -e "Incorrect protocol name specified, please use: <IP> <PORT> <http/ftp>"

exit;;
esac
exit 0
```

# Script Output:

 ```sh
[root@rhel-100 home]# ./cipherlist-host_port_protocol.sh 192.168.1.1 21 ftp 
Starting OpenSSL TLS scanning for 192.168.1.1:21 

Certificate Subject: CN = Name Common AD 
Certificate Issuer:  CN = CA92 

Supported Server Cipher(s): 
TLSv1.2: ECDHE-RSA-AES256-GCM-SHA384 
TLSv1.2: DHE-RSA-AES256-GCM-SHA384 
TLSv1.2: ECDHE-RSA-AES128-GCM-SHA256 
TLSv1.2: DHE-RSA-AES128-GCM-SHA256 
TLSv1.2: ECDHE-RSA-AES256-SHA384 
TLSv1.2: DHE-RSA-AES256-SHA256 
TLSv1.2: ECDHE-RSA-AES128-SHA256 
TLSv1.2: DHE-RSA-AES128-SHA256 
TLSv1.2: AES256-SHA256
TLSv1.3: TLS-AES-256-GCM-SHA384
TLSv1.3: TLS-CHACHA20-POLY1305-SHA256
TLSv1.3: TLS-AES-128-GCM-SHA256
```

### Todos

 - Make the script work with hostnames
 - Add more failure checks when incorrect ports are used
 
 ### Alternatives (way wayyy better and more complex)

 - Linux - [testssl.sh](https://github.com/drwetter/testssl.sh)
 - Windows - [sslscan](https://github.com/rbsec/sslscan)

License
----

MIT
