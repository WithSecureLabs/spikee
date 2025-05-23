# Creating Custom Targets

Targets in `spikee` are Python scripts that interact with an LLM, API, or guardrail. They define how `spikee test` sends prompts and receives responses.

## Structure

A target script must be placed in the `targets/` directory (either local workspace or built-in) and contain a function named `process_input`.

**Signature:**

```python
from typing import Union, Tuple, Optional, Dict, Any
import os
from dotenv import load_dotenv

# Optional: Load .env for API keys
# load_dotenv()

def process_input(input_text: str, system_message: str = None, logprobs: bool = False) -> Union[str, bool, Tuple[str, Optional[dict]]]:
    """
    Processes the input prompt and returns the result.

    Args:
        input_text (str): The prompt generated by spikee.
        system_message (str, optional): System message, if provided by the dataset.
        logprobs (bool, optional): Hint for targets supporting log probability requests.
                                    The target should handle this gracefully if not supported.

    Returns:
        str: The LLM's text response.
        bool: For guardrail targets: True if bypassed (attack succeeded), False if blocked (attack failed).
        Tuple[str, Optional[dict]]: Advanced targets can return (response_text, logprobs_dict).
                                      Logprobs are not currently used by spikee's core logic
                                      but might be useful for custom analysis or future features.
                                      If logprobs are requested but unavailable, return (response_text, None).
    """
    # --- Implementation ---
    # 1. Initialize client (e.g., OpenAI, Azure) if needed. Use os.getenv() for keys from .env.
    # 2. Format the request using input_text and system_message.
    # 3. Make the API call or model inference request. Handle potential exceptions.
    # 4. Process the response.
    # 5. Return the result in the correct format (str, bool, or tuple).

    # Example (Mock LLM):
    response_text = f"Processed: {input_text[:50]}..."
    # Return simple string or tuple if handling logprobs
    return response_text # Or return (response_text, None)

    # Example (Mock Guardrail - simplistic):
    # is_blocked = "malicious" in input_text.lower()
    # attack_succeeded = not is_blocked # True if attack bypassed, False if blocked
    # return attack_succeeded
```

**Key Points:**

1.  **Location:** Store custom targets in `./targets/your_target_name.py`.
2.  **Function Name:** Must be `process_input`.
3.  **Parameters:** Accepts `input_text`, optional `system_message`, optional `logprobs` hint.
4.  **Return Value:**
    * **LLM Target:** Return the model's response as a `str`. Optionally return `(str, dict)` if handling `logprobs`.
    * **Guardrail Target:** Return a `bool`. `True` means the attack **bypassed** the guardrail (success). `False` means the attack was **blocked** (failure). **This logic is crucial for correct analysis.**
5.  **Error Handling:** Implement robust error handling for API calls (timeouts, rate limits, etc.). `spikee test` includes basic retry logic (`--max-retries`) for 429 errors, but target-specific errors should be handled within the script or raised.
6.  **Dependencies:** Add required libraries to your environment.
7.  **API Keys:** Use `os.getenv("YOUR_API_KEY_ENV_VAR")` to load credentials from the `.env` file.

**Example: Simple API Call (Conceptual)**

```python
# ./targets/my_api_target.py
import os
import requests
from dotenv import load_dotenv

load_dotenv() # Load .env file

API_ENDPOINT = os.getenv("MY_API_ENDPOINT")
API_KEY = os.getenv("MY_API_KEY")

def process_input(input_text, system_message=None, logprobs=False):
    if not API_ENDPOINT or not API_KEY:
        raise ValueError("API Endpoint or Key not configured in .env")

    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Content-Type": "application/json"
    }
    payload = {
        "prompt": input_text,
        "system_prompt": system_message
        # Add other parameters as needed by your API
        # Optionally handle 'logprobs' parameter if API supports it
    }

    try:
        response = requests.post(API_ENDPOINT, headers=headers, json=payload, timeout=60)
        response.raise_for_status() # Raise HTTPError for bad responses (4xx or 5xx)
        result = response.json()
        # Adapt this based on your API's response structure
        response_text = result.get("completion", "")
        # If logprobs were requested and returned by API:
        # logprobs_data = result.get("logprobs", None)
        # return response_text, logprobs_data
        return response_text # Return only text if logprobs not handled/available

    except requests.exceptions.RequestException as e:
        print(f"Error calling My API: {e}")
        # Re-raise the exception to be handled by spikee's tester retry logic
        # Or handle specific errors differently if needed
        raise
```