# peptide_signature_finder

import pandas as pd
from tkinter import Tk
from tkinter.filedialog import askopenfilename
import numpy as np

# Function to select files interactively
def select_file(prompt="Select CSV file"):
    """Interactive file selection dialog"""
    Tk().withdraw()  # Prevents empty Tk window
    return askopenfilename(title=prompt, filetypes=[("CSV Files", "*.csv")])

# Function to read and validate mass spectrometry data
def read_ms_data(filepath):
    """Read and validate mass spectrometry data"""
    try:
        df = pd.read_csv(filepath)
        
        # Ensure required columns exist
        required_columns = ['m/z', 'intensity']
        if not all(col.lower() in df.columns.str.lower() for col in required_columns):
            raise ValueError("CSV must contain 'm/z' and 'intensity' columns")
            
        # Standardize column names
        df.columns = [col.lower().strip() for col in df.columns]
        df = df.rename(columns={'mass': 'm/z'})
        
        return df
    except Exception as e:
        print(f"Error reading file: {str(e)}")
        return None

# Function to match peptides
def match_peptides(sample_data, reference_data, tolerance_ppm=10):
    """Match peptides within specified ppm tolerance"""
    matches = []
    
    for _, ref_row in reference_data.iterrows():
        ref_mz = ref_row['m/z']
        
        # Calculate ppm difference for each sample peak
        sample_data['ppm_diff'] = abs((sample_data['m/z'] - ref_mz) / ref_mz * 1000000)
        
        # Find closest match within tolerance
        closest_match = sample_data[sample_data['ppm_diff'] <= tolerance_ppm].sort_values('ppm_diff')
        
        if not closest_match.empty:
            best_match = closest_match.iloc[0]
            matches.append({
                'peptide_signature': ref_row['peptide_signature'],
                'reference_mz': ref_mz,
                'matched_mz': best_match['m/z'],
                'intensity': best_match['intensity'],
                'ppm_difference': best_match['ppm_diff']
            })
    
    return pd.DataFrame(matches)

# Main analysis function
def analyze_mass_spec_data():
    """Main function to analyze mass spectrometry data"""
    print("Select sample data file...")
    sample_path = select_file("Select Sample Data CSV")
    if not sample_path:
        return
    
    print("Select reference peptide signatures file...")
    ref_path = select_file("Select Reference Peptides CSV")
    if not ref_path:
        return
    
    # Read sample data
    sample_df = read_ms_data(sample_path)
    if sample_df is None:
        return
    
    # Read reference data
    ref_df = pd.read_csv(ref_path)
    if 'peptide_signature' not in ref_df.columns:
        print("Error: Reference file must contain 'peptide_signature' column")
        return
    
    # Perform matching
    tolerance_ppm = float(input("Enter ppm tolerance for matching (default 10): ") or 10)
    matches_df = match_peptides(sample_df, ref_df, tolerance_ppm)
    
    # Save results
    if not matches_df.empty:
        output_path = "filtered_peptides.csv"
        matches_df.to_csv(output_path, index=False)
        print(f"\nResults saved to: {output_path}")
        print("\nMatched peptides:")
        print(matches_df[['peptide_signature', 'reference_mz', 'matched_mz', 'ppm_difference']])
    else:
        print("\nNo matches found within specified tolerance.")

# Run analysis
if __name__ == "__main__":
    analyze_mass_spec_data()
