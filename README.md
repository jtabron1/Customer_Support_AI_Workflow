# Building the Customer Support Email Workflow in n8n

This workflow automatically responds to customer support emails using AI classification and a knowledge base.

## Prerequisites
- n8n account
- Gmail API credentials configured
- OpenRouter API key
- OpenAI API key
- Pinecone vector database account with an "FAQ" namespace

## Step-by-Step Instructions

### Step 1: Create the Workflow Foundation
1. In n8n, click **New** to create a new workflow
2. Name it "My workflow" (or your preferred name)
3. You'll start with a blank canvas

### Step 2: Add Gmail Trigger
1. Click the **+** button to add a node
2. Search for and select **Gmail Trigger**
3. Connect your Gmail credentials (if not already connected, click **Create new credential** and authenticate with your Gmail account)
4. In the trigger settings:
   - Set **Poll times** to "every minute"
   - Keep other settings as default
5. This node will check for new emails every minute

### Step 3: Add Text Classifier
1. Add a new node by clicking **+**
2. Search for **Text Classifier** (from @n8n/n8n-nodes-langchain)
3. Connect it after Gmail Trigger
4. Configure the classifier:
   - Set **Input Text** to `={{ $json.text }}`
   - Add two categories:
     - **Category 1**: "Customer Support" - Description: "An email that is related to helping a customer. They may be asking questions about our policies, product, or services. They may need help resetting their passwords or reactivating their accounts. They may be interested in booking time to speak with me."
     - **Category 2**: "Other" - Description: "Any email that is not customer support related."
5. Connect your OpenRouter Chat Model as the language model (you'll create this next)

### Step 4: Add OpenRouter Chat Model
1. Add a new node and select **OpenRouter Chat Model** (from @n8n/n8n-nodes-langchain)
2. Add your OpenRouter API credentials
3. This will serve as the language model for the text classifier
4. Keep the model selection at default

### Step 5: Add No Operation Node (for Non-Support Emails)
1. Add a **No Operation, do nothing** node
2. This node will handle emails classified as "Other" so they don't receive an automated response
3. Connect the second output of the Text Classifier to this node (for non-customer support emails)

### Step 6: Add AI Agent
1. Add a new node and select **Agent** (from @n8n/n8n-nodes-langchain)
2. Configure the agent:
   - Set **Prompt Type** to "define"
   - Set **Text** to `={{ $json.text }}`
   - In **System Message**, enter:
     ```
     # Overview
     You are a helpful customer support agent for NextGentGRC. Your job is to respond to incoming emails with relevant information using your knowledgeBase tool.
     
     ## Instructions
     - Your output should be friendly and use emojis
     - Sign off as the Agent who's solving tomorrow's GRC problems, today
     
     ## Output
     - Output only the body content of the email
     ```
   - Connect this to receive input from the Text Classifier (first output for customer support emails)

### Step 7: Add OpenRouter Chat Model for Agent
1. Add another **OpenRouter Chat Model** node (this will power the AI Agent)
2. Set the model to `openai/gpt-5-mini`
3. Connect your OpenRouter API credentials
4. This will be used by the AI Agent for generating responses

### Step 8: Add Knowledge Base (Pinecone)
1. Add a new node and select **Vector Store - Pinecone** (from @n8n/n8n-nodes-langchain)
2. Configure it as a tool:
   - Set **Mode** to "retrieve-as-tool"
   - Set **Tool Description** to "Call this tool to access FAQ"
   - Select your Pinecone index (named "ragtest")
   - Set **Namespace** to "FAQ"
3. Connect your Pinecone API credentials
4. This allows the AI Agent to access your knowledge base for accurate responses

### Step 9: Add Embeddings
1. Add a new node and select **Embeddings OpenAI**
2. Connect your OpenAI API credentials
3. This will embed queries to match against your knowledge base

### Step 10: Connect the Knowledge Base to Agent
1. Connect the Embeddings OpenAI node as an input to the Pinecone Knowledge Base node
2. Connect the Knowledge Base node as an AI tool to the AI Agent

### Step 11: Add Gmail Reply Node
1. Add a new node and select **Gmail** (n8n-nodes-base.gmail)
2. Configure it to reply:
   - Set **Operation** to "reply"
   - Set **Message ID** to `={{ $('Gmail Trigger').item.json.id }}`
   - Set **Email Type** to "text"
   - Set **Message** to `={{ $json.output }}`
   - Uncheck **Append Attribution**
3. Connect your Gmail API credentials
4. This node will send the AI-generated response back to the customer

### Step 12: Connect the Full Workflow
1. Connect the AI Agent's output to the Gmail Reply node
2. Verify all connections:
   - Gmail Trigger → Text Classifier
   - OpenRouter Chat Model → Text Classifier (as language model)
   - Text Classifier → AI Agent (first output, for support emails)
   - Text Classifier → No Operation (second output, for other emails)
   - OpenRouter Chat Model1 → AI Agent (as language model)
   - Embeddings OpenAI → Knowledge Base
   - Knowledge Base → AI Agent (as tool)
   - AI Agent → Gmail Reply

### Step 13: Test the Workflow
1. Click **Save** to save your workflow
2. Before activating, test with sample data:
   - Click the Gmail Trigger node and use the test email data provided in the workflow
   - Click **Test workflow** to see if it processes correctly
3. Check that the AI Agent generates an appropriate response

### Step 14: Activate the Workflow
1. Toggle the **Active** switch at the top to turn on the workflow
2. The workflow will now automatically check for new emails every minute and respond to customer support inquiries

## How It Works

1. **Gmail Trigger** checks for new emails every minute
2. **Text Classifier** determines if the email is customer support related
3. For support emails:
   - **AI Agent** processes the email
   - **Knowledge Base** is queried for relevant FAQ information
   - An emoji-friendly response is generated
4. For other emails:
   - **No Operation** node handles them without response
5. **Gmail Reply** sends the response back to the customer

## Tips

- Ensure your Pinecone knowledge base is populated with FAQ content
- Test the workflow with various email types before full activation
- Monitor the workflow executions to verify accuracy
- Update the system message in the AI Agent if you want different response styles
- The workflow uses GPT-5-mini model, which is faster and more cost-effective than larger models
