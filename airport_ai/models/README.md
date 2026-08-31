# Model files (not committed)

Pretrained model weights are deliberately excluded from Git because they are
large and may have separate redistribution terms.

For the current Airport AI integration, copy the locally available artifacts
into this directory before running analysis:

- `best_efficientnet.pt`
- `best_cnn.pt`
- `best_model_svm_rbf.pkl` and associated label/scaler files
- `beats_mlp.pt`
- `BEATs_iter3_plus_AS2M.pt` (the separately obtained BEATs encoder)

The application will report a clear missing-model error if no usable model is
present. Do not commit recordings, AES keys, `.enc` files, or model weights.
