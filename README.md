# Notes

"""
class PageProcessor:
    def __init__(self):
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name='us-east-1'  # Replace with your region
        )
    
    def extract_page_interactions(self, metadata):
        page_interactions = []
        for component in metadata['components']:
            for interaction in component['interactions']:
                actions = interaction['actions'][0]
                if actions.get('destinationId'):
                    page_interactions.append({
                        'sourceId': component['id'],
                        'sourceName': component['name'],
                        'triggerType': interaction['trigger']['type'],
                        'destinationId': actions['destinationId'],
                        'transition': actions.get('transition', None)
                    })
        return page_interactions
"""
"""
Idea is for every page, iteraction metadata is used to identify which componenets have interactions and their ID's/destination. Cluade is given all necessary data and prompts to
output javascript code that holds all the interactions for the whole wireframe and edited html code for the page that interacts with the javascript. (fintuning and changes must be made)

ideas:
1. interactions are iterated through one by one to potentially increase accuracy but will also increase runtime (code above to seperate individual interactions)
2. this program processes the metadata pages one by one while updating a javascript file, but another way this could be done is by using the whole wireframe metadata(more research needs to be done)
"""
import boto3
import json
import base64

class ClaudeSonnetProcessor:
    def __init__(self, region_name='us-east-1'):
        self.bedrock = boto3.client(
            service_name='bedrock-runtime',
            region_name=region_name
        )

    def encode_image_to_base64(self, image_path):
        try:
            with open(image_path, "rb") as image_file:
                return base64.b64encode(image_file.read()).decode('utf-8')
        except FileNotFoundError:
            raise FileNotFoundError(f"Image file not found at path: {image_path}")
    
    def generate_claude_response(self, prompt_text, image_base64):
        try:
            messages = [
                {
                    "role": "user",
                    "content": [
                        {
                            "type": "image",
                            "source": {
                                "type": "base64",
                                "media_type": "image/jpeg",
                                "data": image_base64
                            }
                        },
                        {
                            "type": "text",
                            "text": prompt_text
                        }
                    ]
                }
            ]

            response = self.bedrock.invoke_model(
                modelId="anthropic.claude-3-5-sonnet-20240620-v1:0",  # Claude 3.5 Sonnet v1
                body=json.dumps({
                    "anthropic_version": "bedrock-2023-05-31",
                    "messages": messages  # Removed max_tokens for intensive tasks
                }),
                contentType="application/json",
                accept="application/json"
            )
            
            response_body = json.loads(response['body'])
            return response_body.get('completion', 'No response from Claude')
        
        except Exception as e:
            print(f"Error invoking Claude model: {e}")
            return None


def save_output_to_files(claude_output, html_output_path, js_output_path):
    """
    NEEDS TO BE TESTED
    """
    try:
        # Extract HTML content
        html_start = claude_output.find("<html>")
        html_end = claude_output.find("</html>") + len("</html>")
        html_content = claude_output[html_start:html_end] if html_start != -1 and html_end != -1 else None

        # Extract JavaScript content
        js_start = claude_output.find("<script>")
        js_end = claude_output.find("</script>") + len("</script>")
        js_content = claude_output[js_start:js_end].replace("<script>", "").replace("</script>", "").strip() if js_start != -1 and js_end != -1 else None

        # Save HTML to file
        if html_content:
            with open(html_output_path, "w") as html_file:
                html_file.write(html_content)
            print(f"HTML output saved to: {html_output_path}")
        else:
            print("No HTML content found in Claude's response.")

        # Save JavaScript to file
        if js_content:
            with open(js_output_path, "w") as js_file:
                js_file.write(js_content)
            print(f"JavaScript output saved to: {js_output_path}")
        else:
            print("No JavaScript content found in Claude's response.")

    except Exception as e:
        print(f"Error saving output to files: {e}")



if __name__ == "__main__":
    processor = ClaudeSonnetProcessor(region_name='us-east-1')

    # Define paths for inputs
    html_path = "./input.html"  # Path to the HTML file
    css_path = "./input.css"  # Path to the CSS file
    interactions_path = "./interactions.json"  # Path to the interaction metadata JSON file
    js_path = "./input.js"  # Path to the JavaScript file
    image_path = "./example_image.jpg"  # Path to the JPEG image
    
    # paths for output files:
    html_output_path = "./output.html"  # Path for the generated HTML output
    js_output_path = "./output.js"  # Path for the generated JavaScript output
    
    # Load HTML content from file
    try:
        with open(html_path, "r") as html_file:
            html_code = html_file.read()
    except FileNotFoundError:
        print(f"HTML file not found at path: {html_path}")
        exit(1)

    # Load CSS content from file
    try:
        with open(css_path, "r") as css_file:
            css_code = css_file.read()
    except FileNotFoundError:
        print(f"CSS file not found at path: {css_path}")
        exit(1)

    # Load interaction metadata from JSON file
    try:
        with open(interactions_path, "r") as interactions_file:
            interactions = json.load(interactions_file)
    except FileNotFoundError:
        print(f"Interactions JSON file not found at path: {interactions_path}")
        exit(1)
    except json.JSONDecodeError:
        print(f"Failed to decode JSON from file: {interactions_path}")
        exit(1)

    # Load JavaScript content from file
    try:
        with open(js_path, "r") as js_file:
            js_code = js_file.read()
    except FileNotFoundError:
        print(f"JavaScript file not found at path: {js_path}")
        exit(1)

    # Encode the image to work with claude
    try:
        encoded_image = processor.encode_image_to_base64(image_path)
    except FileNotFoundError as e:
        print(e)
        exit(1)

    prompt_text = f"""

    You are a front-end developer who excels at creating interactions between HTML files using JavaScript 
    and has a great understanding of Figma metadata. You are given 4 inputs and a set of instructions you must follow.

    <instructions>
    Using the Figma interaction metadata for the page, analyze the HTML and CSS of the page and create 
    proper interactions for it in the JavaScript file.
    </instructions>

    <inputs>
    Figma Page Interaction MetaData:
    {json.dumps(interactions, indent=2)}

    HTML:
    {html_code}

    CSS:
    {css_code}

    JAVASCRIPT:

    </inputs>

    <output>
    The expected output is a JSON object that contains:
    1. Edited HTML code that will interact with the JavaScript code.
    2. JavaScript code linking interactable components in the HTML to their respective destination IDs/pages.
    </output>

    Additionally, analyze and use the provided image as context for buttons and possible interactions.
    """

    response = processor.generate_claude_response(prompt_text, encoded_image)

    if response:
        print("Claude's Response:")
        print(response) # will have to split response into html and javascript later.
