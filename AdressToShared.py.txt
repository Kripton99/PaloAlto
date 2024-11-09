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

    # Device group içindeki tüm adres objelerini listeleme
    def get_all_addresses_in_device_group(self, device_group):
        xpath = f"/config/devices/entry[@name='localhost.localdomain']/device-group/entry[@name='{device_group}']/address"
        url = f"{self.host}/api/?type=config&action=get&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        
        addresses = []
        if response.status_code == 200:
            tree = ET.ElementTree(ET.fromstring(response.content))
            root = tree.getroot()
            for entry in root.findall(".//entry"):
                address_name = entry.get("name")
                address_type = entry.find(".//ip-netmask") or entry.find(".//ip-range")
                if address_name and address_type is not None:
                    address_value = address_type.text
                    address_type_tag = address_type.tag
                    addresses.append({"name": address_name, "type": address_type_tag, "value": address_value})
            print(f"{len(addresses)} adet adres objesi bulundu.")
        else:
            print("Adres objeleri alınamadı:", response.text)
        
        return addresses

    # Shared alanda adres objesinin olup olmadığını kontrol etme
    def check_shared_address_exists(self, address_name):
        xpath = f"/config/shared/address/entry[@name='{address_name}']"
        url = f"{self.host}/api/?type=config&action=get&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        return response.status_code == 200 and "<entry" in response.text

    # Adres objesini shared alanda oluşturma (eğer yoksa)
    def create_shared_address(self, address_name, address_type, address_value):
        # Shared alanda adres objesinin varlığını kontrol et
        if self.check_shared_address_exists(address_name):
            print(f"Adres objesi '{address_name}' shared alanda zaten mevcut. Oluşturma işlemi atlandı.")
            return

        # Shared alanda adres objesini oluştur
        xpath = "/config/shared/address"
        element = f"<entry name='{address_name}'><{address_type}>{address_value}</{address_type}></entry>"
        url = f"{self.host}/api/?type=config&action=set&xpath={xpath}&element={element}&key={self.api_key}"
        response = requests.get(url, verify=False)
        if response.status_code == 200:
            print(f"Adres objesi '{address_name}' shared alanda oluşturuldu.")
        else:
            print(f"Adres objesi '{address_name}' shared alanda oluşturulamadı:", response.text)

    # Device group'taki politikaları kontrol edip adres referanslarını güncelleme
    def update_policies_with_shared_addresses(self, device_group, address_name):
        xpath = f"/config/devices/entry[@name='localhost.localdomain']/device-group/entry[@name='{device_group}']/pre-rulebase/security/rules"
        url = f"{self.host}/api/?type=config&action=get&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)

        if response.status_code == 200 and "<entry" in response.text:
            tree = ET.ElementTree(ET.fromstring(response.content))
            root = tree.getroot()
            for rule in root.findall(".//entry"):
                rule_name = rule.get("name")
                rule_xpath = f"{xpath}/entry[@name='{rule_name}']"
                updated = False

                # Source adresleri güncelleme
                for member in rule.findall(".//source/member"):
                    if member.text == address_name:
                        member.text = f"shared/{address_name}"  # shared alanına taşınan adresi referans al
                        updated = True

                # Destination adresleri güncelleme
                for member in rule.findall(".//destination/member"):
                    if member.text == address_name:
                        member.text = f"shared/{address_name}"
                        updated = True

                # Eğer güncelleme yapıldıysa, kuralı güncelle
                if updated:
                    rule_xml = ET.tostring(rule, encoding="unicode")
                    update_url = f"{self.host}/api/?type=config&action=edit&xpath={rule_xpath}&element={rule_xml}&key={self.api_key}"
                    update_response = requests.get(update_url, verify=False)
                    if update_response.status_code == 200:
                        print(f"Kural '{rule_name}' shared adresle güncellendi.")
                    else:
                        print(f"Kural '{rule_name}' güncellenemedi:", update_response.text)

    # Orijinal device group adres objesini silme
    def delete_address_from_device_group(self, device_group, address_name):
        xpath = f"/config/devices/entry[@name='localhost.localdomain']/device-group/entry[@name='{device_group}']/address/entry[@name='{address_name}']"
        url = f"{self.host}/api/?type=config&action=delete&xpath={xpath}&key={self.api_key}"
        response = requests.get(url, verify=False)
        if response.status_code == 200:
            print(f"Adres objesi '{address_name}' device group'tan silindi.")
        else:
            print(f"Adres objesi '{address_name}' device group'tan silinemedi:", response.text)
    def list_device_groups(self):
        API_KEY= self.get_api_key()
        url = f"{PANORAMA_HOST}/api/?type=config&action=get&xpath=/config/devices/entry[@name='localhost.localdomain']/device-group&key={API_KEY}"
        response = requests.get(url, verify=False)
        device_groups = []

        if response.status_code == 200:
            tree = ET.ElementTree(ET.fromstring(response.content))
            root = tree.getroot()
            
            for entry in root.findall(".//device-group/entry"):
                group_name = entry.get("name")
                if group_name:
                    device_groups.append(group_name)
            
            if not device_groups:
                print("Hiçbir device grubu bulunamadı.")
            else:
                print("Device grupları alındı.")
        else:
            print("Device grupları alınamadı:", response.text)

        return device_groups
# Ana İşlem Akışı
PANORAMA_HOST = "https://ipadd"
USERNAME = "username"
PASSWORD = "pass"
api = PanoramaAPI(PANORAMA_HOST, USERNAME, PASSWORD)

# Kullanıcıdan device group seçimi
device_groups = api.list_device_groups()
print("Device Grupları:")
for idx, group in enumerate(device_groups):
    print(f"{idx + 1}. {group}")

device_group_index = int(input("Adres objelerini shared yapacağınız device grubunu seçin (sayı ile): ")) - 1
device_group = device_groups[device_group_index]

# Tüm adres objelerini alma
addresses = api.get_all_addresses_in_device_group(device_group)

# Adres objelerini shared alana taşıma ve politikaları güncelleme
for address in addresses:
    # Adres objesini shared alanda oluştur (varsa atla)
    api.create_shared_address(address["name"], address["type"], address["value"])

    # Politikaları güncelle
    api.update_policies_with_shared_addresses(device_group, address["name"])

    # Orijinal adres objesini device group'tan silme
    api.delete_address_from_device_group(device_group, address["name"])
