```
import hmac
import hashlib
import requests

'''
pip install requests
'''

def frostbit_deactivate(resource_id, action):
  url = "https://2024.holidayhackchallenge.com:443/turnstile"
  challenge_hash = b'6487b8b081bc4317cc8017a898c7dfc8'
  message = f'{resource_id}:{action}'
  h = hmac.new(challenge_hash, message.encode(), hashlib.sha256)
  hash_value = h.hexdigest()
  data = {'rid': resource_id, 'hash': hash_value, 'action': action}
  querystring = {'rid': resource_id}
  try:
      response = requests.post(url, params=querystring, json=data, timeout=5)
      response_data = response.json()
      if response_data.get('result') != 'success':
          print(':(')
          return
      return
  finally:
      pass
      
frostbit_deactivate("76012360-932a-46e9-821d-a723e958380f", "hard")
frostbit_deactivate("76012360-932a-46e9-821d-a723e958380f", "easy")

```