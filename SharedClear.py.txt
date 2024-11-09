import requests
import xml.etree.ElementTree as ET
import urllib3

# SSL uyarılarını gizle
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)

class PanoramaAPI:
    def __init__(self, host, username, password):
        self.host = host
        self.username = username
        self.password = password
        self.api_key = self.get_api_key()

    def get_api_key(self):
        url = f"{self.host}/api/?type=keygen&user={self.username}&password={self.password}"
        response = requests.get(url, verify=False)
        if response.status_code == 200 and "<key>" in response.text:
            api_key = response.text.split("<key>")[1].split("</key>")[0]
            print("API anahtarı alındı.")
            return api_key
        else:
            print("API anahtarı alınamadı:", response.text)
            return None

    # Shared alanda bulunan tüm adres objelerini listeleme
    def get_all_shared_addresses(self):
        xpath = "/config/shared/address"
        url = f"{self.host}/api/?type=config&action=get&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        
        addresses = []
        if response.status_code == 200:
            tree = ET.ElementTree(ET.fromstring(response.content))
            root = tree.getroot()
            for entry in root.findall(".//entry"):
                address_name = entry.get("name")
                if address_name:
                    addresses.append(address_name)
            print(f"{len(addresses)} adet shared adres objesi bulundu.")
        else:
            print("Shared adres objeleri alınamadı:", response.text)
        
        return addresses

    # Belirli bir adres objesinin kullanım durumunu kontrol etme
    def is_address_used(self, address_name):
        xpath = f"/config//member[text()='{address_name}']"
        url = f"{self.host}/api/?type=config&action=get&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        
        # Eğer adres herhangi bir kuralda veya objede kullanılıyorsa, sonuç döner.
        return response.status_code == 200 and "<member>" in response.text

    # Kullanılmayan shared adres objesini silme
    def delete_shared_address(self, address_name):
        xpath = f"/config/shared/address/entry[@name='{address_name}']"
        url = f"{self.host}/api/?type=config&action=delete&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        if response.status_code == 200:
            print(f"Adres objesi '{address_name}' shared alandan silindi.")
        else:
            print(f"Adres objesi '{address_name}' shared alandan silinemedi:", response.text)

# Ana İşlem Akışı
PANORAMA_HOST = "https://panorama_ip"
USERNAME = "username"
PASSWORD = pass"
api = PanoramaAPI(PANORAMA_HOST, USERNAME, PASSWORD)

# Tüm shared adres objelerini al
shared_addresses = api.get_all_shared_addresses()

# Her bir shared adres objesinin kullanım durumunu kontrol et ve kullanılmıyorsa sil
for address_name in shared_addresses:
    if not api.is_address_used(address_name):
        api.delete_shared_address(address_name)
    else:
        print(f"Adres objesi '{address_name}' kullanıldığı için silinmedi.")
