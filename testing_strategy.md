# Testing Strategy: GCP Resource Extractor

## Overview

Testing a GCP resource extraction tool requires careful planning, especially when we need to develop and test without connecting to live GCP environments. This document outlines a comprehensive testing strategy that focuses on mocking GCP services and resources to ensure our code is stable before deploying against actual Google Cloud infrastructure.

## Testing Layers

### 1. Unit Testing

Focus on testing individual components in isolation:

- **Authentication Module**: Test both service account and OAuth authentication flows
- **Resource Discovery Engine**: Test hierarchy traversal logic
- **Extraction Workers**: Test resource configuration extraction
- **Output Formatter**: Test YAML generation and formatting

### 2. Integration Testing

Test interactions between components:

- **Authentication + Resource Discovery**: Test that authenticated clients can discover resources
- **Discovery + Extraction**: Test that discovered resources can be properly extracted
- **Extraction + Output**: Test that extracted configurations are properly formatted and exported

### 3. End-to-End Testing

Simulate full application flows with mocked GCP services.

## GCP Mocking Strategies

### 1. Google Cloud Library Mocking

The Google Cloud Python client libraries support mocking for testing:

```python
from unittest import mock
from google.cloud import storage

def test_list_buckets():
    # Create a mock client
    storage_client = mock.create_autospec(storage.Client)
    
    # Configure the mock to return specific test data
    mock_buckets = [
        mock.Mock(name="bucket1"),
        mock.Mock(name="bucket2")
    ]
    storage_client.list_buckets.return_value = mock_buckets
    
    # Call your function that uses the client
    result = list_storage_buckets(storage_client)
    
    # Verify expected behavior
    assert len(result) == 2
    assert result[0]["name"] == "bucket1"
    storage_client.list_buckets.assert_called_once()
```

### 2. Using `pytest-mock` For Simplified Mocking

```python
def test_list_buckets(mocker):
    # Mock the client and its methods
    mock_storage_client = mocker.patch('google.cloud.storage.Client')
    mock_bucket1 = mocker.Mock(name="bucket1")
    mock_bucket2 = mocker.Mock(name="bucket2")
    
    # Configure return values
    mock_storage_client.return_value.list_buckets.return_value = [mock_bucket1, mock_bucket2]
    
    # Call function under test
    result = list_storage_buckets()
    
    # Assertions
    assert len(result) == 2
```

### 3. Google Cloud Emulator

For some GCP services, local emulators are available:

- **Datastore/Firestore**: `gcloud beta emulators datastore start`
- **Pub/Sub**: `gcloud beta emulators pubsub start`
- **Bigtable**: `gcloud beta emulators bigtable start`

These allow testing against a local service that mimics the behavior of the cloud service.

### 4. MockServer for HTTP API Testing

For REST API interactions, MockServer can simulate GCP API responses:

```python
def test_api_interaction():
    with requests_mock.Mocker() as m:
        m.get('https://compute.googleapis.com/compute/v1/projects/test-project/zones/us-central1-a/instances',
              json={"items": [
                  {"name": "instance-1", "id": "123", "machineType": "n2-standard-2"},
                  {"name": "instance-2", "id": "456", "machineType": "n2-standard-4"}
              ]})
              
        # Call code that uses this API endpoint
        instances = get_compute_instances("test-project", "us-central1-a")
        
        assert len(instances) == 2
        assert instances[0]["name"] == "instance-1"
```

### 5. VCR.py for Recording and Replaying API Interactions

For more complex API interactions, VCR.py can record real API calls and replay them during testing:

```python
import vcr

@vcr.use_cassette('fixtures/gcp_list_instances.yaml')
def test_list_compute_instances():
    # First run will make real API calls and record responses
    # Subsequent runs will use recorded responses
    instances = list_compute_instances("test-project", "us-central1-a")
    
    assert len(instances) > 0
    assert "name" in instances[0]
```

### 6. Custom Mock Data Factory

Create a comprehensive mock data factory for GCP resources:

```python
class GCPMockFactory:
    @staticmethod
    def create_compute_instance(name, machine_type="n2-standard-2", zone="us-central1-a"):
        return {
            "name": name,
            "id": f"instance-{hash(name)}",
            "machineType": machine_type,
            "zone": zone,
            "status": "RUNNING",
            "networkInterfaces": [
                {"network": "default", "accessConfigs": [{"natIP": "34.68.28.219"}]}
            ]
        }
    
    @staticmethod
    def create_storage_bucket(name, location="US"):
        return {
            "name": name,
            "id": f"bucket-{hash(name)}",
            "location": location,
            "storageClass": "STANDARD",
            "timeCreated": "2025-05-01T10:00:00Z"
        }
    
    # Add methods for each resource type
```

## Testing for Error Handling and Edge Cases

### 1. API Error Simulation

```python
def test_rate_limit_handling(mocker):
    # Mock API client to simulate rate limit error
    mock_client = mocker.patch('google.cloud.compute.InstancesClient')
    mock_client.return_value.list.side_effect = [
        google.api_core.exceptions.ResourceExhausted("Rate limit exceeded"),
        [mock.Mock(name="instance-1")]  # Second call succeeds
    ]
    
    # Test that our retry logic handles the rate limit error
    result = list_instances_with_retry(project="test-project")
    
    # Verify the function retried and eventually succeeded
    assert len(result) == 1
    assert mock_client.return_value.list.call_count == 2
```

### 2. Pagination Testing

```python
def test_pagination_handling(mocker):
    # Mock client to return paginated results
    mock_client = mocker.patch('google.cloud.storage.Client')
    page1 = [mocker.Mock(name="bucket1"), mocker.Mock(name="bucket2")]
    page2 = [mocker.Mock(name="bucket3")]
    
    # Configure pagination
    mock_client.return_value.list_buckets.return_value = page1
    mock_client.return_value.list_buckets.side_effect = [
        page1,  # First call returns page 1
        page2   # Second call returns page 2
    ]
    
    # Test function that handles pagination
    result = list_all_buckets()
    
    # Verify all pages were processed
    assert len(result) == 3
```

## Test Data Management

### 1. Sample Resource Fixtures

Create JSON fixtures for each GCP resource type to use in tests:

```
tests/
  fixtures/
    compute_instance.json
    storage_bucket.json
    cloudsql_instance.json
    bigquery_dataset.json
    ...
```

### 2. Mock Response Factory

```python
def create_gcp_list_response(resource_type, items, next_page_token=None):
    """Create a standardized GCP list response."""
    response = {"items": items}
    if next_page_token:
        response["nextPageToken"] = next_page_token
    return response
```

## CI/CD Testing Approach

1. **Development**: Use pure mocks for fast, local testing
2. **CI Pipeline**: Use recorded API responses (VCR) for realistic testing
3. **Pre-Production**: Test with a controlled test GCP project with minimal resources
4. **Production**: Deploy only after thorough testing in previous environments

## Testing Framework and Tools

### Recommended Tools

1. **pytest**: Test framework
2. **pytest-mock**: For simplified mocking
3. **requests-mock**: For HTTP API mocking
4. **VCR.py**: For recording/replaying API calls
5. **coverage.py**: For code coverage analysis

### Sample Test Configuration

```python
# tests/conftest.py
import pytest
import os

@pytest.fixture
def mock_gcp_client():
    """Provide a pre-configured mock GCP client."""
    with mock.patch('google.cloud.storage.Client') as mock_client:
        yield mock_client

@pytest.fixture
def sample_resource_data():
    """Load test fixtures."""
    with open('tests/fixtures/compute_instance.json') as f:
        return json.load(f)
```

## Conclusion

By implementing this comprehensive testing strategy with a focus on mocking GCP services, the GCP Resource Extractor can be thoroughly tested without requiring a live GCP account during development. This approach ensures stable, predictable behavior before deploying against real cloud infrastructure, while also enabling rapid iteration during development.