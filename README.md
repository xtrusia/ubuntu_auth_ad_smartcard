# How to setup AD - SSSD(smartcard)

# Prerequsite

PIV support smartcard

# Setting Windows

## AD DS

```
Install-WindowsFeature -Name AD-Domain-Services -IncludeManagementTools
Import-Module ADDSDeployment
Install-ADDSForest -DomainName "seyeong.kim" -DomainNetbiosName "SEYEONGKIM" -InstallDNS -SafeModeAdministratorPassword (Read-Host "DSRM Password" -AsSecureString) -Force
```

## AD CA
```
Install-WindowsFeature ADCS-Cert-Authority, ADCS-Web-Enrollment -IncludeManagementTools
Install-AdcsCertificationAuthority -CAType StandaloneRootCA -CryptoProviderName "RSA#Microsoft Software Key Storage Provider" -KeyLength 2048 -HashAlgorithmName SHA256 -ValidityPeriod Years -ValidityPeriodUnits 5 -Force
Install-AdcsWebEnrollment -Force
```
## Create User Seyeong


# Ubuntu

## Installations
```
sudo apt install libpam-sss opensc-pkcs11 pcscd libpam-pkcs11 sssd opensc sssd-dbus sssd-tools sssd-ad realmd adcli
sudo add-apt-repository ppa:yubico/stable && sudo apt-get update
sudo apt upgrade -y && sudo apt install libpam-sss opensc-pkcs11 pcscd libpam-pkcs11 sssd opensc sssd-dbus sssd-tools yubico-piv-tool -y
sudo apt install libengine-pkcs11-openssl libp11-3 opensc libssl-dev -y
sudo apt install yubikey-manager ykcs11 -y
```
## Make sure dns relove
netplan setting.
```
echo "nameserver 172.16.132.141" >> /etc/resolv.conf ; chattr +i /etc/resolv.conf
```

## Check realm
```
sudo realm -v discover WIN-E7RF5O29H6J.seyeong.kim
sudo realm -v join WIN-E7RF5O29H6J.seyeong.kim
```
## generate 
```
ykman piv reset -f
yubico-piv-tool -a generate -s 9a -A RSA2048
export PKCS11_MODULE_PATH=/usr/lib/x86_64-linux-gnu/libykcs11.so
openssl req  -engine pkcs11 -keyform engine   -new   -key "pkcs11:object=Private key for PIV Authentication;type=private;pin-value=123456"   -config openssl.cnf -reqexts v3_req   -out seyeongkim.csr -multivalue-rdn
cat seyeongkim.csr
```
# Windows

Request cert with CSR(seyeongkim.csr)
download it ( say it is seyeongkim.der )

## upload certificate for User
save it windows server
```
$cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2("seyeongkim.der")
Set-ADUser -Identity 'Seyeong' -Certificates @{ Add = $cert }
```
# Ubuntu
```
openssl x509 -in seyeongkim.der -out seyeongkimissued.pem -outform pem

cat seyeongkimissued.pem |  openssl x509 -text -noout

yubico-piv-tool -a import-certificate -s 9a -i seyeongkimissued.pem

pkcs15-tool --read-certificate 1 > card-cert.pem
openssl x509 -text -noout -in card-cert.pem

openssl x509 -in certnew.cer -out cacert.crt -outform pem

sudo rm /usr/local/share/ca-certificates/cacert.crt
sudo rm /etc/ssl/certs/cacert.pem
sudo cp cacert.crt /usr/local/share/ca-certificates/
sudo update-ca-certificates

mkdir -p /etc/sssd/pki/
sudo rm /etc/sssd/pki/sssd_auth_ca_db.pem
sudo touch /etc/sssd/pki/sssd_auth_ca_db.pem
cat cacert.crt | sudo tee -a /etc/sssd/pki/sssd_auth_ca_db.pem

openssl verify -verbose card-cert.pem
openssl verify -verbose -CAfile /etc/sssd/pki/sssd_auth_ca_db.pem card-cert.pem

systemctl restart sssd

sudo dbus-send --system --print-reply --dest=org.freedesktop.sssd.infopipe /org/freedesktop/sssd/infopipe/Users org.freedesktop.sssd.infopipe.Users.ListByCertificate string:"$(cat card-cert.pem)" uint32:10
```

## setup conf files
/etc/sssd/sssd.conf
/etc/pam.d/common-auth
/etc/krb5.conf
