
## Common Test 1.2

### Dataset Preprocessing and Tokenization Rationale

This repository contains a data preprocessing pipeline for a dataset comprised of expressions in the form:

```
event type : Feynman diagram : amplitude : squared amplitude
```

### Overview

The goal of the preprocessing step is to prepare the amplitudes and squared amplitudes for downstream tasks (such as sequence-to-sequence modeling) by:

- **Tokenizing** the input sequences in a structured way.
- **Normalizing** indices within the expressions to ensure consistency.
- **Splitting** the aggregated dataset into training, validation, and test sets using an 80-10-10 split.

The data is split across 10 files within a ZIP archive, and our pipeline processes each file to produce tokenized JSON outputs for each dataset split.

### Tokenization Strategy

Our tokenization approach is built around a regular expression that aims to capture all meaningful parts of the mathematical expressions:

```python
r'([A-Za-z_]+|\d+\.\d+|\d+|[+\-*/^(){}\[\]:,])'
```

This pattern was chosen with several key considerations in mind:

- **Identifier Handling:**  
  The `[A-Za-z_]+` segment captures words and identifiers. This is essential for retaining variable names, function names, and other symbolic elements that often include underscores.

- **Numerical Precision:**  
  We explicitly capture both integers and floating-point numbers using `\d+\.\d+` and `\d+`. This ensures that numerical components of the expressions are tokenized accurately, preserving their inherent precision.

- **Operators and Punctuation:**  
  The inclusion of specific mathematical operators (`+`, `-`, `*`, `/`, `^`) and punctuation (such as parentheses, braces, brackets, colons, and commas) ensures that the syntactic structure of the mathematical expressions is maintained. This explicit splitting allows the model to understand and work with the structure of the expressions rather than treating them as undifferentiated text.

### Normalization of Indices

Throughout the dataset, indices in the form of `_123456` can appear and may vary in value. To handle this, the function `normalize_indices` is employed:

- **Purpose:**  
  Normalizes indices by replacing each unique index (e.g., `_123456`) with a sequential identifier (e.g., `_1`, `_2`, etc.) based on the order of appearance.

- **Benefits:**  
  - **Vocabulary Control:** Reduces the vocabulary size by preventing a proliferation of unique index tokens.
  - **Consistency:** Helps the model focus on the underlying mathematical structure rather than on arbitrary numerical differences.
  - **Generalization:** Improves the robustness of the model by normalizing variations that do not affect the semantics of the expressions.

### Design Considerations

- **Granularity vs. Structure:**  
  Our approach strikes a balance between fine-grained tokenization (isolating individual operators, numbers, and punctuation) and preserving contextual information (such as entire variable or function names). This balance is critical for enabling effective parsing and understanding of complex mathematical expressions.

- **Scalability and Adaptability:**  
  The regex-based tokenization is both modular and extensible. If the dataset evolves to include additional symbols or more complex structures, the tokenization logic can be easily updated without a complete redesign of the preprocessing pipeline.

- **Modular Preprocessing Pipeline:**  
  The pipeline is organized into clear, modular functions:
  - **`tokenize`**: Breaks down expressions into tokens.
  - **`normalize_indices`**: Standardizes variable indices.
  - **`process_file`**: Reads and processes each line in the dataset files.
  - **`split_dataset`**: Shuffles and splits the processed data into training, validation, and test sets.
  
  This modularity not only aids in debugging and maintenance but also allows for future enhancements as dataset requirements evolve.

---

## Common Task 2


### Transformer Model Training and Evaluation

### Overview

We implement a BERT-style Transformer model designed for masked language modeling (MLM). The pipeline is as follows:

1. **Vocabulary Building:**  
   - We first build a vocabulary from the tokenized amplitude and squared amplitude sequences, including special tokens such as `<pad>`, `<bos>`, `<eos>`, and `<mask>`.
2. **Dataset Preparation:**  
   - The data is loaded from JSON files (previously generated by our preprocessing step), and token sequences from both the amplitude and squared amplitude columns are concatenated with `<bos>` at the beginning and `<eos>` at the end.
   - A custom PyTorch `Dataset` and `DataLoader` manage batching and padding.
3. **Positional Encoding:**  
   - A positional encoding module is integrated into the model. This ensures that the Transformer, which lacks inherent sequential bias, can capture the order of tokens in the sequence.
4. **Masked Language Modeling (MLM):**  
   - A masking function replaces a random subset of tokens (excluding special tokens) with the `<mask>` token. Some tokens are replaced by random tokens or kept unchanged to simulate noise. The MLM objective is to predict these masked tokens, which trains the model to understand context.
5. **Model Architecture:**  
   - Our model consists of an embedding layer, positional encoding, a Transformer encoder (with multiple layers and attention heads), and a linear layer that projects the encoder outputs to the vocabulary space.
6. **Training and Evaluation:**  
   - During training, the model learns to predict the original tokens for masked positions using a cross-entropy loss (ignoring non-masked positions with an `ignore_index` of `-100`).
   - We evaluate the model on a held-out validation set and compute the average loss. Although sequence accuracy is the final evaluation metric, loss serves as a proxy for performance improvement.
7. **Display Predictions:**  
   - A helper function displays sample predictions by replacing masked tokens with model predictions, enabling visual inspection of the model's performance.

### Detailed Design Considerations

- **Vocabulary and Special Tokens:**  
  The vocabulary is constructed by scanning through the tokenized sequences. Special tokens play a critical role in:
  - Marking sequence boundaries (`<bos>` and `<eos>`).
  - Providing padding support (`<pad>`).
  - Enabling the MLM task by marking positions to predict (`<mask>`).

- **Sequence Construction:**  
  Each example is formed by concatenating tokens from both amplitude and squared amplitude sequences. This creates a unified sequence, ensuring the model sees the entire context during training.

- **Positional Encoding:**  
  The positional encoding module uses sinusoidal functions to generate fixed position embeddings. This is crucial for the Transformer model to learn the order of tokens without recurrent structures.

- **Masking Strategy:**  
  The masking function follows a three-step process:
  - **80% Replacement:** Replace masked tokens with `<mask>` tokens.
  - **10% Replacement:** Replace masked tokens with a random token.
  - **10% Retention:** Keep the original token.
  
  This approach (similar to BERT) prevents the model from overfitting to the `<mask>` token while learning a robust representation.

- **Model Architecture:**  
  - **Transformer Encoder:**  
    A bidirectional encoder is used without causal masking. This is ideal for the MLM task where context on both sides of a masked token is needed.
  - **Scalability:**  
    The model parameters (like `d_model`, number of attention heads, and layers) are configurable, allowing for scalability and adaptation to more complex tasks.

- **Training Loop:**  
  The training loop runs over multiple epochs (set to 500 in our example) and performs:
  - **Forward Pass:** Compute predictions from the model.
  - **Loss Calculation:** Use cross-entropy loss, ignoring positions that were not masked.
  - **Backpropagation and Optimization:** Update model parameters using Adam optimizer.
  - **Validation:** Monitor performance on a separate validation set to gauge overfitting and model improvements.

- **Forward-Thinking Considerations:**  
  - **Modular Design:**  
    Each component (vocabulary, dataset, masking, and model architecture) is modular, making it easy to update or replace parts as needed.
  - **Flexibility:**  
    The masking probability and model architecture can be adjusted, supporting experimentation with different configurations.
  - **Interpretability:**  
    By displaying masked token predictions, the design facilitates debugging and understanding of model behavior.

---

## Repository Structure
- **Processed Data:**  
  Processed Train, val, test data in json files done as part of Common Test 1.2.

- **Common_test_2.pth:**  
  File containing weights of trained model done in Common test 2.

- **Common_Task2.ipynb**  
  Jupyter Notebook containing implementation of Common Task 1.2 and Common Task 2.
