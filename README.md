# openssl-cipher-list-scan
A rework of the script found on the https://www.ise.io/using-openssl-determine-ciphers-enabled-server/ page

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
	echo "Starting OpenSSL TLS scanning for $1 $2"
  else
	echo "No connection could be established with $1 $2 (check the provided IP and PORT)"
  exit 1
fi

e=$(for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -starttls ftp -cipher $c -no_tls1_3 < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done)
 
d=$(echo $e | rev | cut -d ' ' -f 1 | rev)

if [ -z "$d" ]; then
  echo -e "\nTLS connection to $1 $2 has failed (make sure that FTPS is enabled)"
else
  
  echo  
		openssl s_client -connect $1:$2 -starttls ftp -cipher $d -no_tls1_3 < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'

  echo -e '\nSupported Server Cipher(s):'
		for v in ssl3 tls1 tls1_1 tls1_2; do
			for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -starttls ftp -cipher $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls+/TLSv/g' | tr '_' '.' 
			done
		done 
  exit 1
fi
;;
	http)
	
if nc -zw1 $1 $2 >/dev/null; then
	echo "Starting OpenSSL TLS scanning for $1 $2"
  else
	echo "No connection could be established with $1 $2 (check the provided IP and PORT)"
  exit 1
fi

e=$(for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
		openssl s_client -connect $1:$2 -cipher $c -no_tls1_3 < /dev/null > /dev/null 2>&1 && echo -e "$c"
	done)
 
d=$(echo $e | rev | cut -d ' ' -f 1 | rev)

if [ -z "$d" ]; then
  echo -e "\nTLS connection to $1 $2 has failed (make sure that HTTPS is enabled)"
else
  
  echo  
		openssl s_client -connect $1:$2 -cipher $d -no_tls1_3 < /dev/null 2>/dev/null | grep 'subject\|issuer' | awk '{sub(/, serialNumber.*/,""); print}' | sed -r 's/subject=+/Certificate Subject: /g' | sed -r 's/issuer=+/Certificate Issuer:  /g'

  echo -e '\nSupported Server Cipher(s):'
		for v in ssl3 tls1 tls1_1 tls1_2; do
			for c in $(openssl ciphers 'ALL:eNULL' | tr ':' ' '); do
				openssl s_client -connect $1:$2 -cipher $c -$v < /dev/null > /dev/null 2>&1 && echo "$v: $c" | sed -r 's/tls+/TLSv/g' | tr '_' '.' 
			done
		done 
  exit 1
fi
;;	

	*)
echo -e "Incorrect protocol name specified, please use: <IP> <PORT> <http/ftp>"

exit;;
esac
exit 0
