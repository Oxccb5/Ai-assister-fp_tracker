-fp_tracker

This code was made by @oxccb5 for Aiassister llm program

Track NFT floor prices and activity to help users make informed decisions on whether a particular NFT is a good deal or not.
import requests
import json
from datetime import datetime, timedelta

class NFTDealEvaluator:
    def __init__(self):
        self.apis = {
            'ethereum': {
                'stats': 'https://api.reservoir.tools/collections/v5',
                'activity': 'https://api.reservoir.tools/events/collections/activity/v6',
                'headers': {'x-api-key': 'YOUR_RESERVOIR_KEY'},
                'params': {'includeTopBid': 'true'}
            },
            'solana': {
                'stats': 'https://api-mainnet.magiceden.io/v2/collections/{}/stats',
                'activity': 'https://api-mainnet.magiceden.io/v2/collections/{}/activities',
                'headers': {'Authorization': 'Bearer YOUR_MAGICEDEN_KEY'}
            },
            'arbitrum': {
                'stats': 'https://arbitrum-api.reservoir.tools/collections/v5',
                'activity': 'https://arbitrum-api.reservoir.tools/events/collections/activity/v6',
                'headers': {'x-api-key': 'YOUR_RESERVOIR_KEY'}
            },
            'base': {
                'stats': 'https://base-api.reservoir.tools/collections/v5',
                'activity': 'https://base-api.reservoir.tools/events/collections/activity/v6',
                'headers': {'x-api-key': 'YOUR_RESERVOIR_KEY'}
            }
        }
        
        self.deal_thresholds = {
            'price_vs_floor': 0.9,
            'volume_spike': 2.0,
            'recent_sales': 5,
            'floor_drop': 0.85
        }
        
        self.session = requests.Session()
        self.session.headers.update({'User-Agent': 'NFTDealEvaluator/1.0'})

    def get_chain_data(self, chain, contract):
        """Improved with proper parameter handling and error logging"""
        try:
            # Validate API configuration
            if chain not in self.apis:
                raise ValueError(f"Unsupported chain: {chain}")
                
            api_config = self.apis[chain]
            
            # Handle different API paradigms
            if chain == 'solana':
                stats_url = api_config['stats'].format(contract)
                activity_url = api_config['activity'].format(contract)
            else:
                stats_url = api_config['stats']
                activity_url = api_config['activity']
                api_config['params'] = {'id': contract}  # Reservoir-style ID param

            # Make validated requests
            stats_response = self._make_api_call(
                chain, 
                stats_url,
                params=api_config.get('params'),
                headers=api_config['headers']
            )
            
            activity_response = self._make_api_call(
                chain,
                activity_url,
                params={'limit': 10, 'sortBy': 'eventTimestamp'} if chain != 'solana' else None,
                headers=api_config['headers']
            )

            return self._normalize_data(chain, stats_response, activity_response)
            
        except Exception as e:
            return {
                'error': str(e),
                'chain': chain,
                'contract': contract,
                'timestamp': datetime.now().isoformat()
            }

    def _make_api_call(self, chain, url, params=None, headers=None):
        """Enhanced API call handler with retries and validation"""
        try:
            response = self.session.get(
                url,
                params=params,
                headers=headers,
                timeout=10
            )
            
            # Log detailed error information
            if response.status_code != 200:
                error_info = {
                    'status': response.status_code,
                    'url': response.url,
                    'response': response.text[:500],  # Truncate long responses
                    'chain': chain,
                    'timestamp': datetime.now().isoformat()
                }
                raise requests.HTTPError(f"API Error: {json.dumps(error_info)}")
                
            return response.json()
            
        except requests.exceptions.RequestException as e:
            raise RuntimeError(f"Network error: {str(e)}") from e

    def _normalize_data(self, chain, stats, activity):
        """Improved data normalization with validation"""
        normalized = {
            'floor_price': 0.0,
            'volume_24h': 0.0,
            'recent_sales': [],
            'chain': chain,
            'valid': False
        }

        try:
            if chain == 'solana':
                # Magic Eden normalization
                normalized['floor_price'] = stats.get('floorPrice', 0) / 1e9
                normalized['volume_24h'] = stats.get('volume24hr', 0) / 1e9
                normalized['recent_sales'] = [{
                    'price': sale['price']/1e9,
                    'time': datetime.fromtimestamp(sale['blockTime']).isoformat(),
                    'difference': (sale['price']/1e9) / (stats.get('floorPrice', 1e-9)/1e9) - 1
                } for sale in activity if sale.get('type') == 'buyNow'][:10]
            else:
                # Reservoir normalization
                collection_data = stats.get('collections', [{}])[0]
                stats_data = collection_data.get('volume', {})
                
                normalized['floor_price'] = float(collection_data.get('floorAskPrice', 0))
                normalized['volume_24h'] = float(stats_data.get('1day', 0))
                normalized['recent_sales'] = [{
                    'price': float(event.get('price', {}).get('amount', {}).get('native', 0)),
                    'time': event.get('event', {}).get('createdAt'),
                    'difference': float(event.get('price', {}).get('amount', {}).get('native', 0)) 
                                / normalized['floor_price'] - 1 if normalized['floor_price'] else 0
                } for event in activity.get('events', []) if event.get('type') == 'sale'][:10]

            normalized['valid'] = any(sale['price'] > 0 for sale in normalized['recent_sales'])
            
        except (KeyError, IndexError, TypeError) as e:
            normalized['error'] = f"Data normalization error: {str(e)}"
            
        return normalized

    # Rest of the class remains similar but uses the improved data structure

# Example debug usage
if __name__ == "__main__":
    evaluator = NFTDealEvaluator()
    
    test_collections = {
        'ethereum': '0xbc4ca0eda7647a8ab7c2061c2e118a18a936f13d',
        'solana': 'mad_lads'
    }
    
    for chain, contract in test_collections.items():
        print(f"\nTesting {chain.upper()} collection: {contract}")
        data = evaluator.get_chain_data(chain, contract)
        
        if 'error' in data:
            print(f"Error: {data['error']}")
        else:
            print(f"Floor Price: {data['floor_price']}")
            print(f"Recent Sales: {len(data['recent_sales'])} valid entries")
            print(f"Data Valid: {data['valid']}")
     from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

retry_strategy = Retry(
    total=3,
    backoff_factor=1,
    status_forcelist=[429, 500, 502, 503, 504]
)
self.session.mount('https://', HTTPAdapter(max_retries=retry_strategy))
from jsonschema import validate

RESERVOIR_SCHEMA = {
    "type": "object",
    "properties": {
        "collections": {"type": "array"},
        "events": {"type": "array"}
    },
    "required": ["collections"]
}

# In _normalize_data:
if chain != 'solana':
    validate(instance=stats, schema=RESERVOIR_SCHEMA)
    def _make_api_call(self, chain, url, params=None, headers=None):
    response = self.session.get(...)
    print(f"Rate limits: {response.headers.get('x-ratelimit-remaining')}")
    return response
