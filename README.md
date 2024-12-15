# Pull-the-lyrics
ดึงเนื้อเพลงจากเว็ป Lyrics 
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from webdriver_manager.chrome import ChromeDriverManager
from tqdm import tqdm  # Progress bar library
import time
import re  # For cleaning filenames

def wait_for_manual_authentication(driver):
    """Pauses the script and waits for the user to manually complete authentication."""
    print("Authentication required. Please complete the verification in the browser.")

    input("Press Enter after completing authentication...")

def clean_filename(filename):
    """Clean the filename by removing or replacing invalid characters."""
    return re.sub(r'[\\/*?:"<>|]', '', filename)

def scrape_lyrics_with_titles(driver, urls):
    lyrics_collection = {}

    for url in tqdm(urls, desc="Scraping Progress"):
        driver.get(url)
        time.sleep(0.5)  # Wait for the page to load

        # Check for the presence of the authentication message
        try:
            auth_message = driver.find_element(
                By.XPATH, "//h1[contains(text(), \"Let's confirm you are human\")]"
            )
            if auth_message:
                wait_for_manual_authentication(driver)
        except Exception:
            # No authentication message, continue
            pass

        try:
            # Extract the song title
            title_element = driver.find_element(By.CSS_SELECTOR, '[data-testid="lyric.title"]')
            title = title_element.text.strip()

            # Extract the lyrics
            lyrics_elements = driver.find_elements(By.CSS_SELECTOR, '[data-testid="lyrics.lyricLine"]')
            lyrics = [elem.text for elem in lyrics_elements if elem.text]

            # Combine the title and lyrics
            lyrics_collection[title] = "\n".join(lyrics)

        except Exception as e:
            lyrics_collection[url] = f"Failed to scrape: {e}"

    return lyrics_collection

if __name__ == "__main__":
    # List of URLs to scrape
    urls = [
        "https://lyrics.lyricfind.com/lyrics/ink-waruntorn-phob-rak",
        "https://lyrics.lyricfind.com/lyrics/boyd-kosiyabong-rak-khun-kao-laew",
        "https://lyrics.lyricfind.com/lyrics/risa-narisa-my-daisy",
    ]
    
    # Set up Selenium with Chrome
    chrome_options = Options()
    chrome_options.add_argument("--disable-gpu")
    chrome_options.add_argument("--no-sandbox")

    driver = webdriver.Chrome(service=Service(ChromeDriverManager().install()), options=chrome_options)

    try:
        # Scrape lyrics and titles
        lyrics_collection = scrape_lyrics_with_titles(driver, urls)

        # Save each song's lyrics to a separate file and print file name with title
        for idx, (title, lyrics) in enumerate(lyrics_collection.items(), start=1):
            # Format the filename with cleaned title
            cleaned_title = clean_filename(title)  # Clean the title for use as a filename
            formatted_title = f"{str(idx).zfill(2)}: {title}"  # Add prefix with leading zeros
            print(formatted_title)  # Print the formatted title

            # Save lyrics to a file
            filename = f"{cleaned_title}.txt"  # Use the cleaned title as the filename
            with open(filename, "w", encoding="utf-8") as file:
                file.write(f"{formatted_title}\n\n{lyrics}")  # Include the title at the top of the file

        print("Lyrics have been saved as separate files.")

    finally:
        # Close the browser
        driver.quit()

