# Aircraft Journey Extractor

Prototype backend extraction path for converting an image of an aircraft journey summary form into structured JSON.

This repository is intentionally small for the take-home assessment. The working artifact is a Colab-oriented notebook, [Aircraft_OCR.ipynb](Aircraft_OCR.ipynb), that loads a sample form image, runs OCR with a vision-language model, and parses the model output into a fixed JSON schema for downstream systems.

## Colab

> **Run the prototype in Google Colab**
>
> [Open Aircraft_OCR.ipynb in Colab](https://colab.research.google.com/drive/1o7qrIgxXuCXiArtA4KZuEDTcVcLXl9gq?usp=sharing)
>
> Recommended runtime: **GPU**

The Colab notebook contains the full image-to-JSON extraction path and is the easiest way to reproduce the sample output.

## Repository Contents

- [Aircraft_OCR.ipynb](Aircraft_OCR.ipynb): minimal extraction notebook.
- [assets/aircraft_journey_summary.jpg](assets/aircraft_journey_summary.jpg): provided sample input image.

## Objective

Given 1-3 aircraft journey summary images, extract common operational fields:

- Aircraft Model
- Registration Number
- Departure Airport
- Arrival Airport
- Passenger
- Crew
- Fuel
- Load
- Defect Message
- Missing Info

The output is normalized into a Python dictionary / JSON-compatible object. Missing fields are represented as `null` and listed in `Missing Info`.

## Approach

The prototype uses `lightonai/LightOnOCR-2-1B` through Hugging Face Transformers.

The extraction flow is:

1. Load the image with `PIL`.
2. Run the image through `LightOnOcrForConditionalGeneration`.
3. Ask the model to produce a structured transcription of the form as an HTML table.
4. Parse the HTML table with `BeautifulSoup`.
5. Map values into a fixed schema.
6. Preserve multi-line defect text by converting HTML line breaks into newline characters.
7. Add `Missing Info` for expected fields that were not found.

This keeps the prototype simple while still handling the hardest part of the sample: handwritten free-text defect notes.

## Key Design Choices

### OCR-specialized vision model

I initially tested OpenAI GPT vision models, but the defect message was less accurate on the provided sample. The defect field is the most operationally important free-text field and is also the least structured, so I prioritized a model that performed better on OCR-style document transcription.

`LightOnOCR-2-1B` was selected because it is specialized for OCR/document understanding and can run in a Colab GPU environment without requiring a hosted API.

### HTML table as intermediate output

The model output is parsed as an HTML table instead of free-form text. This gives the notebook a small but reliable bridge between model output and JSON:

- table rows map naturally to field/value pairs;
- `BeautifulSoup` handles HTML parsing robustly;
- line breaks in defect text can be preserved;
- missing expected fields can be detected after parsing.

### Fixed schema

The downstream contract is kept explicit. Every expected field is present in the output even if missing from the image.

## Assumptions

- Input images are aircraft journey summary forms with broadly similar layout and field names.
- Images are readable enough for OCR after normal loading; no advanced preprocessing is currently applied.
- Airport fields are expected as short airport codes when present.
- Numeric fields such as passenger, crew, fuel, and load are returned as strings in this prototype to avoid accidentally changing units or handwritten notation.
- If a field is absent or not confidently extracted, it should remain `null` and appear in `Missing Info`.
- The notebook is a prototype artifact, not a production API server.

## Scope and Constraints

In scope:

- Single-image extraction.
- 1-3 sample images by rerunning the notebook cells or adapting the image loading cell.
- Structured JSON output.
- Missing-field reporting.
- Multi-line defect message preservation.

Out of scope:

- Production deployment.
- Authentication, persistence, queues, or database writes.
- Batch upload API implementation.
- Human review workflow.
- Confidence scores.
- Fine-tuning.
- Advanced image preprocessing such as deskewing, denoising, field segmentation, or handwriting-specific correction.

## Running the Notebook

1. Open [Aircraft_OCR.ipynb](Aircraft_OCR.ipynb) in Google Colab.
2. Enable GPU runtime.
3. Run the dependency installation cells:

```python
!pip install transformers
!pip install pillow pypdfium2
```

4. Run all cells.
5. The final cell prints the extracted JSON-compatible dictionary.

The current notebook loads the sample image from:

```python
https://raw.githubusercontent.com/yylam-ai/Aircraft-OCR/main/assets/aircraft_journey_summary.jpg
```

For local or private images, replace the image loading cell with a file upload or local path.

## Sample Output

For [assets/aircraft_journey_summary.jpg](assets/aircraft_journey_summary.jpg), the notebook produces:

```json
{
  "Aircraft Model": "Airbus A320",
  "Registration Number": "9M-XX1",
  "Departure Airport": "KVL",
  "Arrival Airport": "SIN",
  "Passenger": null,
  "Crew": "4",
  "Fuel": "12k",
  "Load": "150",
  "Defect Message": "var7 90ml DD 244161 and\nDD 244162, AC 655 P200\nCONTROL and AC 655 9000\nPD 60 (F007) LT TO BE\nUNCHECKED. OPERATE",
  "Missing Info": [
    "Passenger"
  ]
}
```

## API Sketch

The current deliverable is a notebook, but the extraction logic can be wrapped behind a small HTTP API.

### Request

`POST /extract`

Content type: `multipart/form-data`

Fields:

- `files`: one or more image files, up to 3 per flight.

Example:

```bash
curl -X POST http://localhost:8000/extract \
  -F "files=@assets/aircraft_journey_summary.jpg"
```

### Response

```json
{
  "documents": [
    {
      "filename": "aircraft_journey_summary.jpg",
      "status": "completed",
      "data": {
        "Aircraft Model": "Airbus A320",
        "Registration Number": "9M-XX1",
        "Departure Airport": "KVL",
        "Arrival Airport": "SIN",
        "Passenger": null,
        "Crew": "4",
        "Fuel": "12k",
        "Load": "150",
        "Defect Message": "var7 90ml DD 244161 and\nDD 244162, AC 655 P200\nCONTROL and AC 655 9000\nPD 60 (F007) LT TO BE\nUNCHECKED. OPERATE",
        "Missing Info": [
          "Passenger"
        ]
      }
    }
  ]
}
```

### Error Response

```json
{
  "documents": [
    {
      "filename": "bad-upload.png",
      "status": "failed",
      "error": {
        "code": "EXTRACTION_FAILED",
        "message": "Unable to extract structured fields from image."
      }
    }
  ]
}
```

## Future Improvements

- Add a second LLM normalization layer after OCR parsing to reconcile field-name variations such as `Passenger`, `Passenger Count`, or similar labels into the fixed JSON schema.
- Add field-level confidence scores.
- Add image preprocessing for skewed, blurry, or low-contrast uploads.
- Add validation for airport codes, registrations, and numeric fields.
- Add a review queue for ambiguous defect messages.
- Add a FastAPI wrapper around the notebook logic.
- Evaluate multiple OCR/document models against a small labelled test set.
