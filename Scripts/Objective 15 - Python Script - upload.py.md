```
#!/usr/bin/python3
import asyncio
import sys
import os
from datetime import datetime, timezone
import aiohttp
import json
import re
import chardet
import socket
import netifaces
import requests
import time
import argparse

def get_ip():
    for interface in netifaces.interfaces():
        addrs = netifaces.ifaddresses(interface)
        if netifaces.AF_INET in addrs:  # IPv4 addresses exist
            for addr in addrs[netifaces.AF_INET]:
                ip = addr['addr']
                if not ip.startswith('127.'):  # Skip loopback
                    return ip
    return None

# Parse command line arguments
parser = argparse.ArgumentParser(description='Upload logs to Elasticsearch')
parser.add_argument('--logs', default='all_logs.log', help='Path to the log file to process (default: all_logs.log)')
parser.add_argument('--host', default='localhost', help='Elasticsearch host IP address (default: localhost)')
args = parser.parse_args()

# Configuration
ELASTIC_HOST = f'http://{args.host}:9200'
ELASTIC_USER = 'elastic'
ELASTIC_PASSWORD = 'ELFstackLogin!'
BATCH_SIZE = 10000
INDEX_NAME = "logs"
rfc_pattern = r"<(?P<facility>\d+)>(?P<version>\d+) (?P<timestamp>[\d\-T:.+]+) (?P<hostname>\S+) (?P<app_name>\S+) (?P<proc_id>-) (?P<msg_id>-) (?P<structured_data>-)(?:\s+(?P<message>.+))?"

def wait_for_elastic(uri=ELASTIC_HOST, user=None, password=None, delay=2):
    url = f"{uri}/_cluster/health"
    auth = (user, password) if user and password else None
    
    attempt = 1
    while True:
        try:
            response = requests.get(url, auth=auth)
            if response.status_code == 200:
                health = response.json()
                status = health.get('status')
                break
        except requests.exceptions.ConnectionError:
            print(f"Attempt {attempt}: Elasticsearch not ready yet...")
            attempt += 1
            time.sleep(delay)

wait_for_elastic(uri=ELASTIC_HOST, user=ELASTIC_USER, password=ELASTIC_PASSWORD)
print("ElasticSearch is ready - running import script")

def detect_file_encoding(file_path):
    """Detect the encoding of a file and default to UTF-8 if necessary"""
    with open(file_path, 'rb') as f:
        raw_data = f.read(10000)  # Read a portion of the file
    result = chardet.detect(raw_data)
    encoding = result['encoding']
    # Default to UTF-8 if ASCII is detected, as ASCII is a subset of UTF-8
    if encoding.lower() == 'ascii':
        encoding = 'utf-8'
    return encoding

def extract_json_and_rfc(line: str) -> tuple:
    """Extract both JSON and RFC standard text from a line"""
    try:
        # Find the first { and last } to extract JSON
        start = line.find('{')
        end = line.rfind('}') + 1
        
        if start == -1 or end == 0:
            return None, line  # No JSON found, treat entire line as RFC text
            
        # Extract JSON and RFC parts
        json_str = line[start:end]
        rfc_text = line[:start].strip()
        
        # Parse JSON
        json_obj = json.loads(json_str)
        
        return json_obj, rfc_text.strip()
    except json.JSONDecodeError:
        return None, line

def extract_rfc_fields(rfc_text: str) -> dict:
    """Extract fields from RFC text"""
    match = re.match(rfc_pattern, rfc_text)
    if match:
        fields = match.groupdict()
        
        # Convert facility and severity from the combined code
        try:
            combined = int(fields['facility'])
            # Syslog facility is the higher bits (shifted right by 3)
            fields['facility'] = combined >> 3
            # Severity is the lower 3 bits
            fields['severity'] = combined & 0x7
        except (ValueError, TypeError):
            pass
            
        # Convert timestamp to proper datetime format
        try:
            dt = datetime.fromisoformat(fields['timestamp'])
            fields['timestamp'] = dt.isoformat()
        except (ValueError, TypeError):
            pass
            
        return fields
    return {}

def process_document(doc: dict, rfc_text: str = "") -> dict:
    """Process document and ensure fields are properly formatted"""
    processed = {
        '@timestamp': datetime.now(timezone.utc).isoformat(),
        'data_stream': {
            'namespace': 'default',
            'type': 'logs',
            'dataset': 'generic'
        },
        '@version': '1',
        'type': 'syslog',
        'tags': ['match'],
        'host': {'ip': '172.18.0.5'}
    }
    
    # Extract RFC fields if present
    if rfc_text:
        rfc_fields = extract_rfc_fields(rfc_text)
        if rfc_fields:
            processed.update({
                'hostname': rfc_fields.get('hostname'),
                'event_source': rfc_fields.get('app_name'),
                'log': {
                    'syslog': {
                        'severity': {
                            'code': rfc_fields.get('severity'),
                            'name': get_severity_name(rfc_fields.get('severity'))
                        },
                        'facility': {
                            'code': 1,
                            'name': 'user-level'
                        }
                    }
                }
            })
    
    # Create event object
    event = {}
    
    # Track if we've found a valid date field
    found_valid_date = False
    event_received_time = None  # Track EventReceivedTime if found
    
    # First pass to find EventReceivedTime
    if 'EventReceivedTime' in doc and isinstance(doc['EventReceivedTime'], str):
        try:
            if ' ' in doc['EventReceivedTime']:
                dt = datetime.strptime(doc['EventReceivedTime'], '%Y-%m-%d %H:%M:%S')
                event_received_time = dt.strftime('%Y-%m-%dT%H:%M:%S.000Z')
            elif 'Z-' in doc['EventReceivedTime'] or 'Z+' in doc['EventReceivedTime']:
                clean_value = doc['EventReceivedTime'].replace('Z', '')
                dt = datetime.fromisoformat(clean_value)
                utc_dt = dt.astimezone(timezone.utc)
                event_received_time = utc_dt.strftime('%Y-%m-%dT%H:%M:%S.%fZ')
            else:
                event_received_time = doc['EventReceivedTime']
        except ValueError:
            pass
    
    # Process all fields into event structure
    for key, value in doc.items():
        # Handle date fields
        if key in ['EventTime', 'Date', 'EventReceivedTime', 'TimeCreated_SystemTime', 'timestamp', 'Received_Time', 'timestamp_start']:
            if isinstance(value, str):
                try:
                    if ' ' in value:
                        dt = datetime.strptime(value, '%Y-%m-%d %H:%M:%S')
                        formatted_time = dt.strftime('%Y-%m-%dT%H:%M:%S.000Z')
                    elif 'Z-' in value or 'Z+' in value:
                        clean_value = value.replace('Z', '')
                        dt = datetime.fromisoformat(clean_value)
                        utc_dt = dt.astimezone(timezone.utc)
                        formatted_time = utc_dt.strftime('%Y-%m-%dT%H:%M:%S.%fZ')
                    else:
                        formatted_time = value
                    
                    # If this is EventTime and we have EventReceivedTime, use EventReceivedTime
                    if key == 'EventTime' and event_received_time:
                        event[key] = event_received_time
                    else:
                        event[key] = formatted_time
                    
                    # Set @timestamp based on priority
                    if event_received_time:
                        processed['@timestamp'] = event_received_time
                        found_valid_date = True
                    elif key in ['EventTime', 'Date', 'timestamp', 'TimeCreated_SystemTime', 'Received_Time', 'timestamp_start']:
                        processed['@timestamp'] = formatted_time
                        found_valid_date = True
                except ValueError:
                    event[key] = value
                    
        # Handle other fields as before...
        elif key == 'Keywords':
            if isinstance(value, (int, float)):
                event[key] = str(value)
            elif isinstance(value, str):
                event[key] = value.replace('0x', '')
            else:
                event[key] = "0"
        # Fields that should stay at root level
        elif key in ['hostname', 'event_source']:
            processed[key] = value
        # Handle special fields
        elif key == 'Opcode':
            event[key] = str(value)
        elif key == 'Protocol':
            event[key] = str(value)
        elif key == 'Type':
            event[key] = str(value)
        elif key in ['Weight', 'AdditionalInformation_Weight']:
            event[key] = str(value)
        elif key in ['ErrorCode', 'Status', 'ReturnCode', 'CryptographicOperation_ReturnCode']:
            event[key] = str(value)
        elif key in ['NetworkInformation_SourcePort', 'NetworkInformation_DestinationPort', 
                    'NetworkInformation_Port', 'IpPort', 'SourcePort', 'DestPort']:
            if value == '-' or value == '':
                event[key] = None
            else:
                try:
                    event[key] = int(value)
                except (ValueError, TypeError):
                    event[key] = None
        # Skip lowercase processId
        elif key == 'ProcessId':
            continue
        # All other fields go into event object
        else:
            event[key] = value
    
    if event:
        processed['event'] = event
    
    # Check if we found a valid date field
    if not found_valid_date:
        print("ERROR: No valid date field (EventTime or Date) found in record:")
        print(json.dumps(doc, indent=2))
        sys.exit(1)
    
    return processed

def get_severity_name(code):
    """Map severity code to name"""
    severity_map = {
        0: 'emergency',
        1: 'alert',
        2: 'critical',
        3: 'error',
        4: 'warning',
        5: 'notice',
        6: 'informational',
        7: 'debug'
    }
    return severity_map.get(code, 'unknown')

async def send_batch(session, batch, batch_num):
    """Send a batch of documents to Elasticsearch"""
    if not batch:
        return 0

    bulk_body = ""
    for doc in batch:
        action = {"index": {"_index": INDEX_NAME}}
        bulk_body += json.dumps(action) + "\n"
        bulk_body += json.dumps(doc) + "\n"

    try:
        async with session.post(
            f"{ELASTIC_HOST}/_bulk",
            data=bulk_body,
            headers={'Content-Type': 'application/x-ndjson'},
            auth=aiohttp.BasicAuth(ELASTIC_USER, ELASTIC_PASSWORD)
        ) as response:
            result = await response.json()
            
            if 'errors' in result and result['errors']:
                print(f"\nBatch {batch_num} had errors:")
                for i, item in enumerate(result['items']):
                    index_result = item.get('index', {})
                    if index_result.get('status') != 201:
                        error = index_result.get('error', {})
                        print(f"\nDocument failed with status {index_result.get('status')}:")
                        print(f"Error type: {error.get('type')}")
                        print(f"Error reason: {error.get('reason')}")
                        print(f"Failed document:")
                        print(json.dumps(batch[i], indent=2))
                        
            successful_docs = sum(1 for item in result['items'] 
                                if item.get('index', {}).get('status') == 201)
            
            return successful_docs
            
    except Exception as e:
        print(f"Error sending batch {batch_num}: {e}")
        return 0

async def delete_and_create_index(session):
    """Delete existing index and create a new one with proper mappings"""
    try:
        async with session.delete(
            f"{ELASTIC_HOST}/{INDEX_NAME}",
            auth=aiohttp.BasicAuth(ELASTIC_USER, ELASTIC_PASSWORD)
        ) as response:
            print(f"Deleting old index: {await response.text()}")
    except Exception:
        pass

    # Then create index with mappings, including the field limit
    settings = {
        "settings": {
            "number_of_shards": 1,
            "number_of_replicas": 0,
            "index.mapping.total_fields.limit": 20000  # Increased to handle Windows Event logs
        },
        "mappings": {
            "dynamic_templates": [
                {
                    "strings_as_keywords": {
                        "match_mapping_type": "string",
                        "mapping": {
                            "type": "keyword"
                        }
                    }
                }
            ],
            "properties": {
                "@timestamp": {"type": "date"},
                "event": {
                    "properties": {
                        "EventTime": {
                            "type": "date",
                            "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||epoch_millis||basic_date_time_no_millis"
                        },
                        "Level": {"type": "keyword"},
                        "EventReceivedTime": {
                            "type": "date",
                            "format": "strict_date_optional_time||yyyy-MM-dd HH:mm:ss||yyyy-MM-dd'T'HH:mm:ss.SSSSSSZ||epoch_millis"
                        },
                        "Keywords": {"type": "keyword"},
                        "EventType": {"type": "keyword"},
                        "SeverityValue": {"type": "long"},
                        "Severity": {"type": "keyword"},
                        "EventID": {"type": "long"},
                        "SourceName": {"type": "keyword"},
                        "ProviderGuid": {"type": "keyword"},
                        "Version": {"type": "long"},
                        "Task": {"type": "long"},
                        "OpcodeValue": {"type": "long"},
                        "Opcode": {"type": "keyword"},
                        "OpcodeDisplayNameText": {"type": "keyword"},
                        "RecordNumber": {"type": "long"},
                        "ProcessID": {"type": "long"},
                        "ThreadID": {"type": "long"},
                        "Channel": {"type": "keyword"},
                        "Category": {"type": "keyword"},
                        "NetworkInformation_SourcePort": {
                            "type": "long",
                            "null_value": None
                        },
                        "NetworkInformation_DestinationPort": {
                            "type": "long",
                            "null_value": None
                        },
                        "IpPort": {
                            "type": "long",
                            "null_value": None
                        },
                        "ProcessInformation_ProcessID": {
                            "type": "keyword"
                        },
                        "DetailedAuthenticationInformation_KeyLength": {
                            "type": "long"
                        },
                        "Protocol": {"type": "keyword"},
                        "Weight": {"type": "keyword"},
                        "AdditionalInformation_Weight": {"type": "keyword"},
                        "Type": {"type": "keyword"},
                        "ErrorState": {"type": "long"},
                        "ErrorCode": {"type": "keyword"},
                        "Status": {"type": "keyword"},
                        "ReturnCode": {"type": "keyword"},
                        "CryptographicOperation_ReturnCode": {"type": "keyword"},
                        "*": {
                            "type": "keyword"
                        }
                    }
                },
                "log": {
                    "properties": {
                        "syslog": {
                            "properties": {
                                "severity": {
                                    "properties": {
                                        "code": {"type": "integer"},
                                        "name": {"type": "keyword"}
                                    }
                                },
                                "facility": {
                                    "properties": {
                                        "code": {"type": "integer"},
                                        "name": {"type": "keyword"}
                                    }
                                }
                            }
                        }
                    }
                },
                "hostname": {"type": "keyword"},
                "event_source": {"type": "keyword"}
            }
        }
    }

    try:
        async with session.put(
            f"{ELASTIC_HOST}/{INDEX_NAME}",
            json=settings,
            auth=aiohttp.BasicAuth(ELASTIC_USER, ELASTIC_PASSWORD)
        ) as response:
            print(f"Creating new index: {await response.text()}")
    except Exception as e:
        print(f"Error creating index: {e}")
        os._exit(1)


async def upload_logs(file_path):
    """Upload logs directly to Elasticsearch"""
    # First count total lines
    stats = {
        'total_lines': sum(1 for _ in open(file_path, 'r', encoding=detect_file_encoding(file_path))),
        'valid_json': 0,
        'process_failures': 0,
        'batch_send_failures': 0,
        'successful_docs': 0,
        'skipped_docs': 0
    }
    
    current_batch = []
    error_samples = []
    
    try:
        async with aiohttp.ClientSession() as session:
            await delete_and_create_index(session)

            # Detect file encoding
            encoding = detect_file_encoding(file_path)
            print(f"Detected file encoding: {encoding}")
           
            batch_num = 0
            processed_lines = 0
            with open(file_path, 'r', encoding=encoding) as file:
                for line_num, line in enumerate(file, 1):
                    try:
                        # Extract both JSON and RFC text
                        doc, rfc_text = extract_json_and_rfc(line)
                        if not doc:
                            continue
                        
                        stats['valid_json'] += 1  # Count valid JSON docs here
                        
                        # Process document with RFC text
                        processed_doc = process_document(doc, rfc_text)
                        current_batch.append(processed_doc)
                        processed_lines += 1
                        
                        # Send batch if full
                        if len(current_batch) >= BATCH_SIZE:
                            print(f"\nSending batch {batch_num} ({len(current_batch)} documents)")
                            docs_sent = await send_batch(session, current_batch, batch_num)
                            if docs_sent < len(current_batch):
                                stats['batch_send_failures'] += (len(current_batch) - docs_sent)
                            stats['successful_docs'] += docs_sent
                            
                            # Print detailed progress
                            print(f"\rProcessed {line_num}/{stats['total_lines']} lines "
                                  f"({(line_num/stats['total_lines'])*100:.2f}%) - "
                                  f"Valid JSON: {stats['valid_json']} - "
                                  f"Successful: {stats['successful_docs']}", end='', flush=True)
                            
                            current_batch = []
                            batch_num += 1
                            
                    except Exception as e:
                        stats['process_failures'] += 1
                        if stats['process_failures'] < 5:
                            error_samples.append(f"Process failure line {line_num}: {str(e)}")
                        continue

                # Send final batch
                if current_batch:
                    print(f"\nSending final batch ({len(current_batch)} documents)")
                    docs_sent = await send_batch(session, current_batch, batch_num)
                    if docs_sent < len(current_batch):
                        stats['batch_send_failures'] += (len(current_batch) - docs_sent)
                    stats['successful_docs'] += docs_sent

            print("\n\nProcessing Summary:")
            print(f"Total lines in file: {stats['total_lines']}")
            print(f"Valid JSON objects found: {stats['valid_json']}")
            print(f"Successfully indexed: {stats['successful_docs']}")
            print(f"Process failures: {stats['process_failures']}")
            print(f"Batch send failures: {stats['batch_send_failures']}")
            print(f"Missing documents: {stats['valid_json'] - stats['successful_docs']}")
            
            if error_samples:
                print("\nSample Errors:")
                for sample in error_samples:
                    print(sample)

    except Exception as e:
        print(f"\nError during upload: {e}")
        raise

    return stats['successful_docs']


async def main():
    log_file = args.logs

    if not os.path.exists(log_file):
        print(f"Error: File not found - {log_file}")
        os._exit(1)

    start_time = datetime.now()
    try:
        total_docs = await upload_logs(log_file)
        duration = datetime.now() - start_time
        print(f"\nUpload completed successfully:")
        print(f"Total documents processed: {total_docs}")
        print(f"Time taken: {duration}")
        print(f"Average speed: {total_docs / duration.total_seconds():.2f} docs/second")
        os._exit(0)
    except Exception as e:
        print(f"\nUpload failed: {e}")
        os._exit(1)

if __name__ == '__main__':
    try:
        asyncio.run(main())
    except:
        pass
    finally:
        os._exit(0)
```