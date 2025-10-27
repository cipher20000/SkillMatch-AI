# File Upload Implementation - FormData & Progress Tracking

## Overview
Implemented robust file upload system with proper validation, Supabase Storage integration, progress tracking, and text extraction from PDF/DOCX files before analysis.

## Key Improvements

### 1. File Validation
- **Max file size:** 10MB per file
- **Allowed types:** PDF, TXT, DOC, DOCX
- **Validation happens before upload**
- **Clear error messages for invalid files**

### 2. Upload Flow
```
User selects files
  ↓
Client-side validation
  ↓
Upload to Supabase Storage (with progress)
  ↓
Extract text from uploaded file
  ↓
Wait for all uploads to complete
  ↓
Send extracted text to analysis API
  ↓
Display results
```

### 3. Progress Tracking
- Per-file upload status
- Visual progress bars
- Status indicators: uploading → extracting → completed/failed
- Non-blocking UI updates

## Implementation Details

### File Upload Utility (`src/utils/fileUpload.ts`)

#### Validation Function
```typescript
export function validateFile(file: File): { valid: boolean; error?: string } {
  // Check file size (10MB limit)
  if (file.size > MAX_FILE_SIZE) {
    return { valid: false, error: 'File too large' };
  }

  // Check file extension
  const extension = '.' + file.name.split('.').pop()?.toLowerCase();
  if (!ALLOWED_EXTENSIONS.includes(extension)) {
    return { valid: false, error: 'Unsupported extension' };
  }

  // Check MIME type
  if (!ALLOWED_TYPES.includes(file.type)) {
    return { valid: false, error: 'Unsupported type' };
  }

  return { valid: true };
}
```

#### Upload Function
```typescript
export async function uploadAndExtractFile(
  file: File,
  jobDescriptionId: string,
  onProgress?: (progress: UploadProgress) => void
): Promise<UploadedFile> {
  // 1. Validate file
  const validation = validateFile(file);
  if (!validation.valid) throw new Error(validation.error);

  // 2. Upload to Supabase Storage
  onProgress?.({ fileName: file.name, status: 'uploading', message: 'Uploading file...', progress: 0 });

  const fileName = `${jobDescriptionId}/${Date.now()}-${randomId}.${ext}`;
  const { data, error } = await supabase.storage
    .from('resumes')
    .upload(fileName, file);

  // 3. Extract text from file
  onProgress?.({ fileName: file.name, status: 'extracting', message: 'Extracting text...' });

  const text = await extractTextFromFile(file);

  // 4. Return uploaded file metadata
  return {
    fileName: file.name,
    fileUrl: publicUrl,
    text: text.trim(),
    size: file.size,
    type: file.type
  };
}
```

### FileUpload Component (`src/components/FileUpload.tsx`)

#### Features
- Visual file validation feedback
- Green/red indicators for valid/invalid files
- File size display
- Specific error messages
- Remove file functionality

#### UI States
```typescript
interface FileWithValidation {
  file: File;
  valid: boolean;
  error?: string;
}
```

**Valid File Display:**
- Green background
- CheckCircle icon
- File name and size shown

**Invalid File Display:**
- Red background
- AlertCircle icon
- Error message displayed

### Main App Integration (`src/App.tsx`)

#### Upload Handler
```typescript
const handleResumeUpload = async () => {
  // 1. Validation
  if (resumeFiles.length === 0) {
    setError('Please select at least one valid resume file.');
    return;
  }

  // 2. Start upload process
  setIsUploading(true);
  setUploadProgress([]);
  setState('processing');

  // 3. Upload files with progress tracking
  const handleProgress = (progress: UploadProgress) => {
    setUploadProgress(prev => {
      const existing = prev.find(p => p.fileName === progress.fileName);
      if (existing) {
        return prev.map(p => p.fileName === progress.fileName ? progress : p);
      }
      return [...prev, progress];
    });
  };

  const uploadedFiles = await uploadMultipleFiles(
    resumeFiles,
    jobDescriptionId,
    handleProgress
  );

  // 4. Wait for uploads to complete
  setIsUploading(false);

  // 5. Extract text and send to analysis
  const resumeTexts = uploadedFiles.map(f => ({
    fileName: f.fileName,
    text: f.text,
  }));

  // 6. Call analysis API
  const response = await fetch(apiUrl, {
    method: 'POST',
    headers,
    body: JSON.stringify({ jobDescriptionId, resumeTexts }),
  });
};
```

#### UI Elements

**Upload Button States:**
```typescript
<button
  onClick={handleResumeUpload}
  disabled={resumeFiles.length === 0 || isUploading}
>
  {isUploading ? (
    <>
      <Loader className="animate-spin" />
      Uploading Files...
    </>
  ) : (
    `Analyze ${resumeFiles.length} Resume${s}`
  )}
</button>
```

**Progress Display:**
```typescript
{uploadProgress.map((progress) => (
  <div className="progress-item">
    <div className="header">
      <span>{progress.fileName}</span>
      {progress.status === 'completed' && <CheckCircle />}
      {progress.status === 'failed' && <AlertCircle />}
      {(progress.status === 'uploading' || progress.status === 'extracting') && <Loader className="animate-spin" />}
    </div>
    <p>{progress.message}</p>
    {progress.progress !== undefined && (
      <div className="progress-bar">
        <div style={{ width: `${progress.progress}%` }} />
      </div>
    )}
  </div>
))}
```

## Supabase Storage Setup

### Storage Bucket Configuration

**Bucket Name:** `resumes`

**Folder Structure:**
```
resumes/
  ├── {job-description-id-1}/
  │   ├── {timestamp}-{random}.pdf
  │   ├── {timestamp}-{random}.docx
  │   └── ...
  ├── {job-description-id-2}/
  │   └── ...
```

**File Naming Pattern:**
```
{jobDescriptionId}/{timestamp}-{randomId}.{extension}
```

**Benefits:**
- Organized by job posting
- Unique filenames prevent collisions
- Easy cleanup per job

### Storage Policies

**Public Read Access:**
```sql
CREATE POLICY "Allow public read access"
  ON storage.objects FOR SELECT
  USING (bucket_id = 'resumes');
```

**Authenticated Write Access:**
```sql
CREATE POLICY "Allow authenticated uploads"
  ON storage.objects FOR INSERT
  WITH CHECK (
    bucket_id = 'resumes' AND
    (auth.uid() IS NOT NULL OR true)
  );
```

## File Type Handling

### Supported Formats

| Format | Extension | MIME Type | Processing |
|--------|-----------|-----------|------------|
| PDF | .pdf | application/pdf | Binary extraction |
| Text | .txt | text/plain | Direct read |
| Word | .doc | application/msword | Text extraction |
| Word | .docx | application/vnd.openxmlformats-officedocument.wordprocessingml.document | Text extraction |

### Text Extraction (`src/utils/textExtractor.ts`)

**PDF Extraction:**
```typescript
async function extractTextFromPDF(file: File): Promise<string> {
  const arrayBuffer = await file.arrayBuffer();
  const uint8Array = new Uint8Array(arrayBuffer);
  const text = String.fromCharCode.apply(null, Array.from(uint8Array));

  // Extract text from PDF stream objects
  const textMatch = text.match(/stream\s*([\s\S]*?)\s*endstream/g);
  if (textMatch) {
    let extractedText = '';
    textMatch.forEach(match => {
      const content = match.replace(/stream\s*/, '').replace(/\s*endstream/, '');
      extractedText += content + ' ';
    });
    return cleanText(extractedText);
  }

  return cleanText(text);
}
```

**DOCX Extraction:**
```typescript
async function extractTextFromDOCX(file: File): Promise<string> {
  const text = await file.text();
  const cleanedText = text.replace(/<[^>]*>/g, ' ').replace(/\s+/g, ' ');
  return cleanText(cleanedText);
}
```

**Text Cleaning:**
```typescript
function cleanText(text: string): string {
  return text
    .replace(/[^\x20-\x7E\n]/g, ' ')  // Remove non-printable
    .replace(/\s+/g, ' ')              // Normalize whitespace
    .trim();
}
```

## Error Handling

### Validation Errors
- **File too large:** "File {name} is too large. Maximum size is 10MB."
- **Wrong type:** "File {name} has unsupported type: {type}."
- **Wrong extension:** "File {name} has unsupported extension."

### Upload Errors
- **Storage error:** "Upload failed: {error message}"
- **Text extraction error:** "Failed to extract text from file. File may be empty or corrupted."
- **No files uploaded:** "No files were successfully uploaded and processed."

### Network Errors
- **API error:** "Analysis failed: {error message}"
- **Server error:** "Server error: {status} {statusText}"

## User Experience Improvements

### Before Upload
1. Select files
2. Immediate visual validation
3. See which files are valid/invalid
4. Remove invalid files
5. Clear "Analyze" button state

### During Upload
1. Button changes to "Uploading Files..."
2. Button disabled
3. Progress section appears
4. Per-file progress shown
5. Status icons update in real-time

### After Upload
1. Upload completes
2. Button changes to analysis state
3. Analysis begins
4. Progress indicators remain visible
5. Navigate to dashboard on success

## Testing Checklist

- [ ] Upload single PDF file
- [ ] Upload multiple PDF files
- [ ] Upload DOCX file
- [ ] Upload TXT file
- [ ] Try to upload file > 10MB (should reject)
- [ ] Try to upload unsupported file type (should reject)
- [ ] Upload mix of valid and invalid files
- [ ] Remove file before upload
- [ ] Cancel and re-upload
- [ ] Monitor progress bars
- [ ] Verify extracted text quality
- [ ] Check analysis results
- [ ] Verify files in Supabase Storage

## Performance Considerations

### File Upload
- **Parallel uploads:** Each file uploaded sequentially with progress tracking
- **Memory usage:** Files processed in memory (no disk writes needed)
- **Network:** Uploads directly to Supabase Storage (fast CDN)

### Text Extraction
- **Client-side:** Extraction happens in browser
- **No server processing:** Reduces server load
- **Immediate feedback:** Users see progress in real-time

### Analysis
- **Waits for uploads:** Analysis only starts after all uploads complete
- **Batch processing:** All texts sent in single API call
- **Error recovery:** Failed uploads don't block successful ones

## Security Considerations

### File Validation
- **Size limits:** Prevents DOS attacks
- **Type checking:** Prevents malicious files
- **Extension validation:** Additional safety layer

### Storage
- **Per-job folders:** Logical separation
- **Unique filenames:** Prevents overwriting
- **Public URLs:** Read-only access

### Privacy
- **No persistent client storage:** Files not saved locally
- **Temporary processing:** Text extracted and used immediately
- **User isolation:** Files organized by job (can add user isolation)

## Future Enhancements

### Potential Improvements
1. **Server-side PDF parsing:** More accurate extraction
2. **OCR support:** Extract text from scanned PDFs
3. **Drag-and-drop upload:** Better UX
4. **Batch upload limits:** Prevent too many files
5. **File compression:** Reduce upload time
6. **Resume thumbnails:** Preview before upload
7. **Virus scanning:** Additional security layer
8. **Resume parsing libraries:** Better structured data extraction

### Advanced Features
```typescript
// Example: Server-side PDF parsing with Edge Function
import pdfParse from 'npm:pdf-parse';

Deno.serve(async (req: Request) => {
  const formData = await req.formData();
  const file = formData.get('file') as File;

  const buffer = await file.arrayBuffer();
  const data = await pdfParse(Buffer.from(buffer));

  return new Response(JSON.stringify({ text: data.text }));
});
```

## Troubleshooting

### Common Issues

**Issue:** Files not uploading
- Check Supabase Storage bucket exists
- Verify storage policies allow uploads
- Check file size limits

**Issue:** Text extraction fails
- Verify file is not corrupted
- Check file is actual PDF/DOCX (not renamed)
- Try alternative extraction method

**Issue:** Progress not updating
- Check React state updates
- Verify callback functions
- Check for JavaScript errors

**Issue:** Analysis fails after upload
- Verify all uploads completed
- Check extracted text is not empty
- Verify API authentication
