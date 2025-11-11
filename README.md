# Cloud-Native-Catalog-Extraction-Pipeline

live demo : []{https://pdf-extractor-frontend-416866646707.us-central1.run.app/}


This tool is a secure, scalable, cloud-native web application designed to automate complex data extraction and reporting tasks. It provides a centralized internal platform for processing large product catalogs and digital assets, transforming them into structured, actionable data in both Excel and Google Sheets formats.

## About The Project

This tool was built to solve the time-consuming and error-prone manual process of transcribing product information from large PDF catalogs and Google Drive folders. The application provides two core functionalities: a "PDF Catalog Extractor" and a "Google Drive Folder Importer."

It is built on a high-availability, serverless architecture that can process multiple, large jobs in parallel, handling complex orchestration, error-checking, and final data aggregation automatically.

## Core Features

* **Scalable PDF Catalog Extraction:** Users can upload multiple, large PDF catalogs. The system intelligently splits the documents into manageable chunks, processes all chunks in parallel to extract text and images, and then aggregates the results into a single, perfectly ordered Excel file and Google Sheet.
* **Intelligent Content Analysis:**
    * **Text Extraction:** Uses the Google Gemini API to analyze page chunks and extract structured product data (Style ID, SKU, Price, Color) into a JSON format.
    * **Image Extraction:** Uses PyMuPDF to extract all product images. It intelligently sorts images based on their visual coordinates (top-to-bottom, left-to-right) on the page to ensure they match the text order.
* **AI-Powered Banner Filtering:** (Optional) Users can enable a "Has Banners" flag, which uses the OpenAI GPT-4o-mini API to perform batch-analysis on extracted images, filtering out non-product images like marketing banners and collages.
* **Direct Google Drive & Sheets Integration:** The application leverages Google's OAuth 2.0 framework to allow users to securely connect their Google accounts.
    * **Google Sheet Output:** For every processed job, the system automatically creates a new Google Sheet in the user's personal Google Drive, populates it with the text data, and inserts all product images using `=IMAGE()` formulas.
    * **Google Drive Input:** Users can skip the PDF step and instead provide one or more Google Drive folder URLs. The system will iterate through the folder, retrieve all images, and generate the same Excel and Google Sheet reports from that content.
* **Robust Error & Rate Limit Handling:** The worker services include exponential backoff and retry mechanisms to gracefully handle API rate limits from both Gemini and OpenAI, ensuring high-volume jobs complete successfully.

## Technical Architecture

The application is built on a decoupled, event-driven microservice architecture, orchestrated by Google Cloud Workflows. This design ensures that each component can scale independently and failures are isolated.

### 1. System Components

* **Frontend:** A React (Vite) single-page application deployed on Google Cloud Run (Public). It handles all user interaction, authentication, and file uploads.
* **Backend API:** A FastAPI (Python) service deployed on Google Cloud Run (Public). This service acts as the application gateway. Its sole responsibilities are user authentication, PDF splitting, and triggering the Cloud Workflow. It does not handle any long-running, heavy processing.
* **Backend Worker:** A FastAPI (Python) service deployed on Google Cloud Run (Private). This is the secure processing engine. It is inaccessible from the public internet and only accepts authenticated requests from the Cloud Workflow. It handles all heavy tasks: calling Gemini, analyzing images with OpenAI, and building the final Excel/Google Sheet files.
* **Orchestrator:** A Google Cloud Workflow serves as the "brain" of the processing pipeline. It manages the state of each job, executing all chunk-processing tasks in parallel (fan-out) and triggering the final aggregation step only after all parallel tasks have successfully completed (fan-in).
* **Database:** Cloud Firestore is used as the single source of truth for all job metadata (status, chunk counts, error messages, final file URLs) and for storing user-specific Google OAuth refresh tokens.
* **File Storage:** Google Cloud Storage (GCS) is the application's "hard drive." It is used for storing the original uploaded PDFs, the temporary 10-page PDF chunks, all intermediate extracted images and JSON, and the final completed Excel files.

### 2. Authentication & Security

The application uses three distinct layers of security:

* **User Authentication (Firebase):** All frontend access is secured by Firebase Authentication. The frontend sends a Firebase JWT to the backend with every request.
* **User Authorization (Google OAuth 2.0):** To access a user's private Google Drive/Sheets, the app uses a standard OAuth 2.0 flow to obtain and securely store a `refresh_token` in Firestore, allowing the worker to act on the user's behalf.
* **Service-to-Service Security (IAM):** All communication between Google Cloud services is secured using IAM Service Accounts and OIDC tokens. The Cloud Workflow, for example, invokes the private worker using a service account that has the "Cloud Run Invoker" role.

### 3. Key Architectural Flows

#### Flow 1: PDF Catalog Extraction

This is the primary "fan-out, fan-in" workflow.

1.  **Upload:** A user uploads a 100-page PDF. To bypass the 32MB Cloud Run request limit, the Frontend first asks the Backend for a GCS Signed URL. The frontend then uploads the file directly to GCS.
2.  **Trigger:** The frontend calls the `/process-pdf` endpoint on the Backend.
3.  **Split:** The Backend downloads the PDF from GCS, splits it into ten 10-page chunks (using PyMuPDF), uploads these chunks back to GCS, and creates a "parent job" in Firestore with `status: 'splitting'`.
4.  **Orchestrate (Fan-Out):** The Backend triggers the Cloud Workflow, passing it the list of 10 chunk URLs. The Workflow then makes 10 parallel calls to the Worker's `/process-chunk` endpoint.
5.  **Process (In Parallel):** Ten instances of the Worker spin up. Each one:
    * Downloads its assigned 10-page chunk.
    * Calls Gemini to get the text data.
    * Calls PyMuPDF and (optionally) OpenAI to get the filtered image paths.
    * Saves its results (e.g., `chunk_1.json`, `chunk_1_img_01.png`) to a unique folder in GCS.
    * Increments the `processed_chunks` counter in Firestore.
6.  **Aggregate (Fan-In):** The Cloud Workflow waits for all 10 parallel tasks to complete. It then makes one final call to the Worker's `/aggregate-results` endpoint.
7.  **Build:** The aggregation worker:
    * **Fetches Data:** Downloads all `chunk_N.json` files and all image files from GCS.
    * **Builds Excel:** Concatenates all JSON data into a `pandas.DataFrame` and creates the final Excel file, embedding the (now globally-sorted) images into the correct rows.
    * **Builds Sheet:** Uploads all images to the user's Google Drive, creates a new Google Sheet, and populates it with the text data and the new `=IMAGE(drive_url, 4, 120, 120)` formulas.
    * **Cleans Up:** Deletes all temporary chunks and files from GCS and updates the Firestore job with `status: 'completed'` and the final file URLs.
8.  **View:** The Frontend, which is polling Firestore, sees the "completed" status and displays the "Download" and "Go to Sheets" buttons to the user.

#### Flow 2: Google Drive Import

This is a simpler, single-task flow that reuses the aggregation components.

1.  **Trigger:** The user pastes a Google Drive folder link in the Frontend, which calls the `/process-drive-folder` endpoint on the Backend.
2.  **Dispatch:** The Backend creates a new job in Firestore and dispatches a single task to the Worker's `/process-drive-job` endpoint.
3.  **Process:** The Worker:
    * Uses the user's `refresh_token` to initialize the Google Drive API.
    * Lists all images in the specified folder.
    * Downloads all images locally.
    * Creates a `pandas.DataFrame` from the image names.
    * Jumps straight to the aggregation logic: builds the Excel file and Google Sheet (using the Drive links it already has).
    * Updates the job to `completed`.
