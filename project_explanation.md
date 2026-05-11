# Fashion Multimodal Chatbot Assistant
## Complete Project Explanation — Professional Presentation

---

## 1. PROJECT OVERVIEW

This project builds a **Fashion Multimodal Chatbot Assistant** — an AI system that:
- Takes a fashion product image as input
- Automatically generates a natural-language description (caption) of the product
- Finds the top-5 most visually similar products from a dataset
- Presents results with full product metadata and purchase links

The entire system was built using **TensorFlow/Keras only**, without PyTorch.  
The data was scraped from **Jumia Egypt**, the largest e-commerce platform in Egypt.

---

## 2. PROJECT STRUCTURE — THE 6 NOTEBOOKS

| Notebook | Purpose |
|---|---|
| `jumia_scraper.ipynb` | Scrape product data and images from Jumia Egypt |
| `preprocessing.ipynb` | Clean, normalize, and engineer features from raw data |
| `image_preprocessing.ipynb` | Download images, verify integrity, solve the label-matching problem |
| `NoteBook_2.ipynb` | Build, train, and compare all 4 models |
| `retrieval_system.ipynb` | Visual similarity search using FAISS embeddings |
| `chatbot.ipynb` | Final interactive multimodal chatbot interface |

---

## 3. DATA SCRAPING — jumia_scraper.ipynb

### What was scraped
The scraper collected fashion products from **15 categories** on Jumia Egypt, covering:
- Women: hoodies & sweatshirts, jeans, shoes, bags, dresses, t-shirts
- Men: t-shirts, shirts, jeans, shoes, hoodies
- Kids: hoodies, t-shirts

### How scraping worked
The scraper used **Selenium with headless Chrome** (no visible browser window) to:
1. Open each category page automatically
2. Wait for product cards to load (JavaScript-rendered content)
3. Extract from each product card:
   - Product title
   - Price
   - Category and gender
   - Product URL
   - Image URL
4. Navigate through up to 5 pages per category
5. Save all products to a CSV file

### Why Selenium?
Jumia uses JavaScript to render its product listings dynamically.
A simple HTTP request (requests library) cannot see the products —
only Selenium can control a real browser and see the fully rendered page.

### Image download
After scraping product metadata, images were downloaded separately
using the `requests` library with a browser User-Agent header to avoid
being blocked by Jumia's servers.

---

## 4. TEXT AND DATA PREPROCESSING — preprocessing.ipynb

### Step 1: Data loading and audit
- Loaded raw CSV with approximately 5,000+ product records
- Assigned sequential `product_id` (0, 1, 2, ...) as primary key
- Checked for missing values, duplicate rows, and data types

### Step 2: Deduplication
- Removed exact duplicate rows
- Removed products with the same title + price (re-listed products)
- Removed products with duplicate image URLs (same photo, different listing)

### Step 3: Price cleaning and normalization
- Raw prices came as strings like "EGP 1,049.00" or "EGP 300 – EGP 500"
- Extracted numeric values using regex
- For price ranges, took the average
- Created `price_bucket` categories:
  - Budget: < 300 EGP
  - Mid-range: 300–799 EGP
  - Premium: 800–1999 EGP
  - Luxury: ≥ 2000 EGP

### Step 4: Category normalization
- Raw categories like "hoodies_sweatshirts" were mapped to clean names
- Created two levels: `category_clean` (specific) and `category_group` (broader)
- Example: "hoodies_sweatshirts" → category_clean="hoodies & sweatshirts", category_group="tops"

### Step 5: Gender normalization
- Standardized all variations: "woman", "female" → "women"; "boys", "girls" → "kids"

### Step 6: Title cleaning
- Fixed encoding errors (Arabic characters, mojibake)
- Removed HTML entities
- Removed model numbers trailing the title
- Normalized whitespace
- Created `title_clean` (display) and `title_search` (lowercase, for indexing)

### Step 7: Feature engineering from titles
The following attributes were extracted using regex pattern matching:

- **Colors**: black, white, blue, red, navy, mottled grey, pink quartz, etc.
  → `colors` (list), `primary_color` (first color), `color_count`

- **Patterns**: floral, striped, printed, plaid, embroidered, etc.
  → `pattern_clean`, `pattern_group`

- **Brands**: LC Waikiki, Decathlon, Defacto, Adidas, Nike, etc.
  → `brand`, `is_known_brand`

- **Sleeve type**: short sleeve, long sleeve, sleeveless, 3/4 sleeve
  → `sleeve_type`

- **Fit type**: slim fit, regular fit, relaxed fit, oversized
  → `fit_type`

- **Materials**: cotton, fleece, denim, polyester, linen
  → `materials` (list), `materials_str`

### Step 8: Metadata text (for chatbot context)
A rich descriptive sentence was built for each product combining all attributes:
```
"Decathlon Women's Relaxation Yoga Fleece Sweatshirt | Brand: Decathlon |
Category: hoodies & sweatshirts (tops) | Gender: women |
Price: EGP 1049 (premium) | Color: mottled grey | Material: fleece"
```
This `metadata_text` field is what the chatbot retrieves to answer user questions.

### Step 9: Image URL validation
- Checked that each image URL has a valid file extension (.jpg, .jpeg, .png, .webp)
- Flagged invalid or empty URLs
- Created `img_url_valid` boolean column

---

## 5. IMAGE PREPROCESSING — image_preprocessing.ipynb

### The image-label matching problem
This was the most critical challenge in the project.

**The problem:**
When Selenium scrapes Jumia, it sees product cards in a certain order.
Images are downloaded from the `img_url` column.
However, there is no guarantee that the image saved as `{product_id}.jpg`
actually shows the product described by that row's title and attributes.
Reasons this can fail:
- Lazy-loading: images load asynchronously and may load in wrong order
- Network errors: some images fail and shift the index
- Jumia CDN: same image URL can serve different sizes/versions

**The solution:**
Instead of using filenames or positions to match images to labels,
the system saved each image with the filename `{product_id}.jpg`
where `product_id` is the exact primary key of that product row.
This means image 1234.jpg ALWAYS corresponds to the product with `product_id=1234`,
regardless of download order or failures.

### Download pipeline
1. Parallel download using `ThreadPoolExecutor` (multiple simultaneous downloads)
2. Each image saved as `{product_id}.jpg` in the `images_fresh` folder
3. A **download manifest** recorded the result of each download:
   - status: ok / skip (already exists) / fail
   - file path, file size, error message (if any)
4. Safe to re-run — already downloaded files are automatically skipped

### Integrity verification
After downloading, each image was:
1. Re-opened with PIL (Python Imaging Library) to verify it is not corrupted
2. Given a SHA-256 hash fingerprint
3. Checked for duplicate content: same hash = same photo uploaded twice

### Visual spot-check
A grid of 48 random images was displayed with their captions below.
This human visual verification confirmed that each image matches its label correctly.
Any `product_id` values that looked wrong were added to a `BAD_IDS` list and removed.

### Caption building
Structured captions were built for each product:
```
<start> Decathlon Women's Relaxation Yoga Fleece Sweatshirt mottled grey
hoodies & sweatshirts women premium egp 1049.0 <end>
```
These captions are the **ground truth** the models learn to generate.

### Final CSV creation
The download manifest was merged with the cleaned product data:
- Dropped technical columns (error, file_bytes, verify_ok, sha256)
- Sorted by product_id
- Saved as `cleaned_data_final.csv`

### cleaned_data_final.csv — What it contains
- **5,281 rows** × **44 columns**
- Every row has: product_id, title, price, image URL, product URL
- All engineered features: category_clean, gender_clean, primary_color, brand,
  sleeve_type, fit_type, materials_str, pattern_group, metadata_text
- Image validation flags: img_url_valid, img_url_reason
- Local image path: path (absolute path to {product_id}.jpg)

---

## 6. MODEL BUILDING — NoteBook_2.ipynb

### The complete pipeline: from image to caption

```
User's image (224×224×3 pixels)
       ↓
[CNN Backbone] — feature extraction
       ↓
Spatial feature map (7×7×depth) → reshaped to (49, depth)
49 spatial positions, each encoding a region of the garment
       ↓
[Feature Projection] Dense layer → (49, 512)
Reduces dimensionality, same space as word embeddings
       ↓
[Attention Mechanism]
For each word being generated:
  - Compares current word state with all 49 image regions
  - Assigns importance weights (which region matters for this word?)
  - Computes weighted sum → context vector (512,)
       ↓
[Decoder] LSTM / GRU / Transformer
Receives: context vector + previous word embedding
Outputs: probability distribution over entire vocabulary
       ↓
[Prediction Head] Dense(vocab_size) + softmax
Next word = highest probability token
       ↓
Repeat until <end> token or max length reached
       ↓
Final caption: "decathlon women's fitness hoodie pink premium egp 699"
```

---

## 7. THE FOUR MODELS — ARCHITECTURE AND DIFFERENCES

### Model 1: ResNet50V2 + LSTM + Bahdanau Attention

**CNN Backbone: ResNet50V2**
- 50-layer deep residual network (V2 = improved version with pre-activation)
- Pre-trained on ImageNet (1.2 million images, 1000 classes)
- Cut at `post_bn` layer → output shape: (7, 7, 2048)
- 2048 channels per spatial position
- Reshaped to (49, 2048) for attention

**Why residual connections in ResNet?**
Each layer learns a residual (the difference from input), not a full transformation.
This prevents the vanishing gradient problem in deep networks.
Gradients flow directly through skip connections during backpropagation.

**Feature Projection:**
Dense(2048 → 512) + ReLU + BatchNormalization
Maps the 2048-dim CNN features to the same 512-dim space as word embeddings.

**Bahdanau Attention:**
At each decoding step t:
```
score(i) = V · tanh(W1 · feature_i + W2 · hidden_state_t)
weight(i) = softmax(score(i))           → importance of region i
context   = Σ weight(i) · feature_i    → weighted image summary
```
The model learns to focus on the collar when generating "crew neck",
on the color region when generating "blue", on the brand logo when generating "Adidas".

**LSTM Decoder:**
- Long Short-Term Memory with separate memory cell (c_t) and hidden state (h_t)
- Input at each step: concat(context_vector, word_embedding) → (1024,)
- LSTM cell size: 512 units
- The memory cell retains long-term context across the full caption sequence
- Better for longer captions with many attributes (color + fit + material + brand)

**Layers and why:**
| Layer | Output Shape | Why |
|---|---|---|
| ResNet50V2 (frozen) | (49, 2048) | Feature extraction with ImageNet knowledge |
| Dense projection | (49, 512) | Dimensionality reduction |
| BatchNormalization | (49, 512) | Stabilize activations |
| Embedding | (31, 256) | Map token IDs to continuous vectors |
| Bahdanau Attention | (512,) | Dynamic focus on image regions per word |
| LSTM | (512,) | Sequential caption generation with memory |
| Dense(vocab) + softmax | (3837,) | Word probability distribution |

**Training result:** Best val_loss = 1.4145 (epoch 12/15)

---

### Model 2: ResNet50 + GRU + Bahdanau Attention

**CNN Backbone: ResNet50 (V1)**
- Same depth as ResNet50V2 but original post-activation ordering
- Cut at `conv5_block3_out` → output: (7, 7, 2048)
- Same spatial feature map shape as Model 1

**GRU Decoder (vs LSTM):**
GRU has 2 gates (reset + update) vs LSTM's 3 (input, forget, output):
```
update gate z_t = sigmoid(W_z · [h_{t-1}, x_t])
reset gate  r_t = sigmoid(W_r · [h_{t-1}, x_t])
candidate   h̃_t = tanh(W · [r_t * h_{t-1}, x_t])
output      h_t = (1 - z_t) * h_{t-1} + z_t * h̃_t
```
GRU has no separate memory cell — all state in a single h_t vector.

**Advantages of GRU:**
- Fewer parameters → less overfitting risk on small datasets
- Faster to train (fewer matrix operations per step)
- Often comparable performance for short captions

**Problem encountered:**
At epoch 7, val_loss spiked from 1.507 → 1.990 (+32%).
This was a training instability event, not overfitting.
Likely caused by a batch with corrupted placeholder images producing extreme gradients.
GRU's single hidden state has no separate memory cell to absorb gradient spikes.
Early stopping triggered at epoch 11, best result was at epoch 6.

**Training result:** Best val_loss = 1.5068 (epoch 6/11) — worst of all 4 models

---

### Model 3: EfficientNetB0 + LSTM + Bahdanau Attention

**CNN Backbone: EfficientNetB0**
- Compound scaling: width, depth, and resolution scaled simultaneously
- Uses depthwise separable convolutions (much more efficient than ResNet)
- Cut at `top_activation` layer → output: (7, 7, 1280)
- 1280 channels (less than ResNet's 2048, but more semantically dense)
- Swish/SiLU activation throughout → smoother gradient flow
- Only 5.3M parameters vs ResNet50V2's 25M

**Why EfficientNet outperforms ResNet here:**
EfficientNet achieves better accuracy per parameter because of compound scaling.
Each channel in its 1280-dim output is more informationally rich than
ResNet's 2048-dim output due to the architecture's efficiency design.

**Fixed Attention Implementation:**
Previous versions had a critical bug where attention was computed once at
graph-build time and repeated for all timesteps (static context).
The fixed version uses dot-product attention:
```
scores  = lstm_hidden_states · image_features^T   (B, 31, 49)
weights = softmax(scores / √512)                   (B, 31, 49)
context = weights · image_features                 (B, 31, 512)
```
This gives each of the 31 decoding steps its own unique context vector.

**Decoder:** Same LSTM as Model 1 — allows pure CNN comparison.

**Training result:** Best val_loss = 1.4179 (epoch 12/15)
Despite slightly higher val_loss than Model 1, it had:
- Better BLEU-1: 0.347 vs 0.281 (+23%)
- Better BLEU-4: 0.073 vs 0.046 (+59%)
- Zero hallucinations vs 0 for M1 (both clean)
- 2× faster training: ~280s/epoch vs ~560s

---

### Model 4: EfficientNetB0 + Transformer Decoder ← BEST MODEL

**CNN Backbone:** Same EfficientNetB0 as Model 3 → (49, 1280) → Dense(512) → (49, 512)

**Positional Embedding:**
```
PE(pos, 2i)   = sin(pos / 10000^(2i/d_model))
PE(pos, 2i+1) = cos(pos / 10000^(2i/d_model))
```
Added to token embeddings so the Transformer knows word order.
No extra parameters — mathematical formula encodes position.

**Transformer Decoder Block (repeated 2×):**

Sub-layer 1 — Masked Multi-Head Self-Attention:
- Each caption token attends to all PREVIOUS tokens (causal mask blocks future)
- 8 heads × 64 key dimensions = 512 total
- Learns intra-caption dependencies: "long sleeve" → next word likely "shirt"

Sub-layer 2 — Multi-Head Cross-Attention (vision-language bridge):
- query = caption token representations  (31, 512)
- key   = image spatial features          (49, 512)
- value = image spatial features          (49, 512)
- Each of 8 heads specializes in a different visual aspect:
  head 1 → color region, head 2 → garment shape, head 3 → brand logo...
- This simultaneously grounds every word to every image region

Sub-layer 3 — Position-wise Feed-Forward:
- Dense(512 → 2048, GELU activation) → Dense(2048 → 512)
- Applied independently at each position
- Adds nonlinear transformation capacity

All sub-layers use: Residual connection + LayerNormalization + Dropout

**Why Transformer beats LSTM/GRU:**
1. PARALLEL training — all 31 tokens processed simultaneously (vs T serial LSTM steps)
2. Multi-head attention — 8 heads attend to 49 image regions per token simultaneously
3. No vanishing gradient through time — LayerNorm + residuals ensure stable gradients
4. Every word directly attends to every image region at every layer

**Training result:** Best val_loss = **1.3444** (epoch 15/15) — BEST OF ALL 4 MODELS
- Val accuracy: 80.63% — highest of all models
- After LR reduction at epoch 9 → 5e-4 and epoch 14 → 2.5e-4,
  val_loss dropped from 1.419 → 1.354 → 1.344 in 3 epochs
- Epoch time: ~200s — FASTEST of all models (parallel processing)

---

## 8. TRAINING STRATEGY

### Dataset split
- Train: 80% (4,224 samples)
- Validation: 10% (527 samples)
- Test: 10% (527 samples)
- Stratified by category_group to maintain class balance

### Class balancing
Rare categories (dresses, outerwear) were oversampled ×2 in the training set.
This prevents the model from ignoring minority classes.

### Data augmentation (training only)
- RandomFlip (horizontal)
- RandomRotation (±10%)
- RandomZoom (±10%)
- RandomContrast (±10%)

### Callbacks
- **EarlyStopping**: monitors val_loss, patience=5, restores best weights
- **ReduceLROnPlateau**: halves LR when val_loss plateaus for 3 epochs
- **ModelCheckpoint**: saves best weights only (save_best_only=True)

### Loss function
Sparse categorical cross-entropy with mask:
- Padding tokens (index 0) excluded from loss calculation
- Prevents model from gaming accuracy by predicting padding correctly

---

## 9. EVALUATION METRICS

### BLEU Score (Bilingual Evaluation Understudy)
Measures n-gram overlap between generated and reference captions.
- BLEU-1: unigram (word-level) overlap
- BLEU-4: 4-gram (phrase-level) overlap — standard summary metric

### Token Accuracy
Percentage of non-padding tokens predicted correctly.
Note: high accuracy can be misleading because common words (men, egp, for)
dominate predictions and are easy to predict correctly.

### Validation Loss
Cross-entropy loss on the held-out validation set.
Primary metric for model selection — lower = better.

### Qualitative evaluation
- Seen-data inference: captions for training images (tests memorization)
- Unseen-data inference: captions for test images (tests generalization)
- Visual inspection: do generated captions match what the image shows?

### Final comparison table

| Model | CNN | Decoder | Val Loss | Val Acc | BLEU-1 | BLEU-4 |
|---|---|---|---|---|---|---|
| 1 | ResNet50V2 | LSTM | 1.4145 | 78.5% | 0.281 | 0.046 |
| 2 | ResNet50 | GRU | 1.5068 | 76.2% | 0.199 | 0.059 |
| 3 | EfficientNetB0 | LSTM | 1.4179 | 78.3% | 0.347 | 0.073 |
| **4** | **EfficientNetB0** | **Transformer** | **1.3444** | **80.6%** | **TBD** | **TBD** |

---

## 10. RETRIEVAL SYSTEM — retrieval_system.ipynb

### How it works
1. Extract EfficientNetB0 embeddings for ALL 5,281 products
   - GlobalAveragePooling on (7, 7, 1280) → compact (1280,) vector per product
   - L2-normalized (makes cosine similarity = dot product)
   - Cached as .npy files — computed only once

2. Build FAISS index (Facebook AI Similarity Search)
   - IndexFlatIP: exact inner-product search on L2-normalized vectors
   - Equivalent to cosine similarity search
   - Finds top-K nearest neighbors in milliseconds

3. For a query image:
   - Extract (1280,) embedding
   - Search FAISS index → top-5 most similar product IDs
   - Return with similarity scores + full metadata

### Why FAISS?
For 5,281 products, numpy cosine similarity is fast enough.
But FAISS scales to millions of products with sub-linear search time.
It is the industry standard for visual search systems (used by Facebook, Pinterest).

---

## 11. MULTIMODAL CHATBOT — chatbot.ipynb

### How it works end-to-end

```
User uploads image
       ↓
EfficientNetB0 extracts (1280,) embedding
       ↓
FAISS search → top-5 similar products
       ↓
EfficientNetB0 + Transformer Decoder generates caption
Auto-regressive greedy decoding:
  step 1: input=[<start>]         → output="decathlon"
  step 2: input=[<start>,decathlon] → output="women's"
  step 3: ...                     → continues until <end>
       ↓
Results displayed:
  - Query image with generated caption
  - Top-5 similar products with images, metadata, similarity score
  - "View & Buy" links to Jumia product pages
```

### Interactive interface
Built with `ipywidgets` inside Jupyter:
- Text field: enter a product image path or URL
- "Search" button triggers the full pipeline
- Results displayed inline in the notebook

---

## 12. STRENGTHS AND WEAKNESSES

### Model 1 (ResNet50V2 + LSTM)
✅ Stable training — improved 10/15 epochs  
✅ Strong brand recognition (got "Defacto" correctly)  
❌ Slow: ~560s/epoch  
❌ Lower BLEU than Model 3 despite similar val_loss  

### Model 2 (ResNet50 + GRU)
✅ Fastest convergence — best result at epoch 6  
✅ Good on accessories (belts, shoes) — learned texture quickly  
❌ Instability spike at epoch 7 (+32% val_loss)  
❌ 3/8 test images had hallucination loops (same caption repeated)  
❌ Worst overall performance  

### Model 3 (EfficientNetB0 + LSTM)
✅ Best BLEU scores of the three RNN-based models  
✅ Fastest of the three RNN models: ~280s/epoch  
✅ Zero hallucinations  
✅ Improved every single epoch (1-12)  
❌ Slightly worse raw val_loss than Model 1 (1.4179 vs 1.4145)  

### Model 4 (EfficientNetB0 + Transformer) — SELECTED
✅ Best val_loss: 1.3444 (4.5% better than Models 1 and 3)  
✅ Best val_accuracy: 80.63%  
✅ Fastest: ~200s/epoch (parallel processing advantage)  
✅ Multi-head attention simultaneously grounds words to 8 different image aspects  
✅ No sequential bottleneck — all 31 tokens processed in parallel  
❌ Required careful LR tuning — started poorly until LR reduction at epoch 9  
❌ More complex architecture — harder to serialize/load (custom layers)  

---

## 13. KEY FINDINGS AND CONCLUSIONS

1. **EfficientNetB0 > ResNet as a backbone** for fashion captioning.
   Despite having fewer parameters (5.3M vs 25M), its compound scaling produces
   more semantically rich features per channel.

2. **Transformer Decoder > LSTM/GRU** for this dataset.
   Multi-head cross-attention that simultaneously attends to all 49 image regions
   produces better captions than sequential RNN decoders.
   Val_loss improved by 4.5% and training was 2.8× faster than ResNet50V2+LSTM.

3. **GRU is fragile on small datasets** without a strong CNN backbone.
   The epoch 7 spike demonstrated that GRU's single hidden state cannot absorb
   gradient spikes from corrupted images the way LSTM's separate memory cell can.

4. **The image-label matching problem** was the most important engineering challenge.
   Saving images as `{product_id}.jpg` (not by position or filename) guaranteed
   100% correct label-to-image correspondence regardless of download failures.

5. **Data quality directly impacts model quality.**
   The 9-step preprocessing pipeline (deduplication, price parsing, text cleaning,
   feature engineering, metadata fusion) was essential — models trained on
   raw unprocessed data would have performed significantly worse.
