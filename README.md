# Embrace-Audio-Processing
This is the repo we play for audio processing related features of Embrace

---------- Main files ---------

Transcripts: Generate csv file from directory of audios (output as "output_transcript")

AudioClone: Main logic to train Tacotron2 and Hifi-gan for audio cloning (with BUGs!!)

AudioProcess: Pipeline to process existing audio generated by human being



-------------------------------

### Audio Cloning ###
1. Fine-Tuning Phase
This phase involves fine-tuning a pre-trained Tacotron2 model using a provided audio sample so that the model can adapt to the specific voice characteristics of the sample.

Steps:
1.1 Prepare Data:

Audio Sample: Ensure the audio sample is in a suitable format (e.g., WAV). You need a transcript of the audio sample (what is being said in the sample).
Create CSV File: This file should contain paths to the audio file and corresponding transcripts. The CSV format might look like this:
--- css ---
ID,Path,Transcript
sample_01,path/to/audio_sample.wav,Text spoken in the audio sample

1.2 Load Pre-trained Tacotron2 Model:

--- python ---
from speechbrain.pretrained import Tacotron2

tacotron2 = Tacotron2.from_hparams(source="speechbrain/tts-tacotron2-ljspeech", savedir="tmpdir_tts")

1.3 DataLoader for Fine-Tuning:

Create a DataLoader that loads your audio sample and its transcript. This step assumes you have implemented a custom dataset that can handle the audio and text pairing.

--- python ---
from torch.utils.data import DataLoader

# Assuming you have a custom dataset class
dataset = YourCustomDataset(csv_file="path/to/csvfile.csv", ...)
data_loader = DataLoader(dataset, batch_size=1, shuffle=True)

1.4 Fine-Tuning Loop:
Implement a training loop that updates the pre-trained Tacotron2 model weights based on the provided sample.

--- python ---
import torch
from torch.optim import Adam

# Assuming the Tacotron2 model is already loaded
optimizer = Adam(tacotron2.parameters(), lr=1e-4)
criterion = torch.nn.MSELoss()  # or other suitable loss functions

for epoch in range(num_epochs):
    for batch in data_loader:
        optimizer.zero_grad()
        
        # Assuming `batch` contains the input text and target Mel-spectrogram
        text_input, mel_target = batch
        
        # Forward pass through Tacotron2
        mel_output, mel_lengths, alignment = tacotron2.encode_batch(text_input)
        
        # Calculate loss
        loss = criterion(mel_output, mel_target)
        
        # Backward pass and optimization
        loss.backward()
        optimizer.step()
        
        print(f"Epoch: {epoch}, Loss: {loss.item()}")

1.5 Save Fine-Tuned Model:

After fine-tuning, save the model parameters for later use.

--- python ---
torch.save(tacotron2.state_dict(), "fine_tuned_tacotron2.pth")


2. Inference Phase
In this phase, you use the fine-tuned Tacotron2 model to generate speech from a new text input, ensuring it mimics the voice from the audio sample.

Steps:
2.1 Load the Fine-Tuned Model:
Load the saved model from the fine-tuning phase.

--- python ---
tacotron2.load_state_dict(torch.load("fine_tuned_tacotron2.pth"))


2.2 Generate Mel-Spectrogram from New Text:
Provide the new text input to the fine-tuned Tacotron2 model.

--- python ---
new_text = ["This is the new text to synthesize."]
mel_output, mel_lengths, alignment = tacotron2.encode_batch(new_text)


2.3 Convert Mel-Spectrogram to Waveform:
Use HiFi-GAN (or any other neural vocoder) to convert the Mel-spectrogram to a waveform.

--- python ---
from speechbrain.pretrained import HifiGan

hifi_gan = HifiGan.from_hparams(source="speechbrain/tts-hifigan-ljspeech", savedir="tmpdir_vocoder")
waveform = hifi_gan.decode_batch(mel_output)

2.4 Save or Play the Generated Audio:
Save the generated waveform as an audio file or play it.

--- python ---
import torchaudio

torchaudio.save("generated_audio.wav", waveform.squeeze(1).cpu(), 22050)

Final Notes:
Fine-Tuning Data: Fine-tuning typically requires more than a single audio sample to adapt the model effectively. If you only have one sample, the results may not be optimal.
Loss Function: Depending on your data, you might want to experiment with different loss functions and learning rates.
Device: Ensure that all computations are being done on the same device (e.g., CPU or GPU).

######

