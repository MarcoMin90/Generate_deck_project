# Generate_deck_project

import os
import json
import zipfile

# Contenuti dei file
cargo_toml = '''[package]
name = "deck-generator-rust"
version = "0.1.0"
edition = "2021"

[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
rand = "0.8"
clap = { version = "4.0", features = ["derive"] }
'''

main_rs = '''use clap::Parser;
use rand::seq::SliceRandom;
use serde::Deserialize;
use std::{fs, path::Path};

#[derive(Parser)]
#[command(author, version, about, long_about = None)]
struct Args {
    /// Numero di carte nel main deck
    #[arg(short, long, default_value_t = 40)]
    main_deck_size: usize,

    /// Numero di carte nell'extra deck
    #[arg(short, long, default_value_t = 15)]
    extra_deck_size: usize,

    /// Numero di carte nel side deck
    #[arg(short, long, default_value_t = 15)]
    side_deck_size: usize,

    /// Usa la combinazione dei pool (true o false)
    #[arg(short, long, default_value_t = false)]
    combine: bool,

    /// Path al file JSON con pool carte
    #[arg(short, long, default_value = "pools_extended.json")]
    pool_file: String,
}

#[derive(Debug, Deserialize, Clone)]
struct Card {
    name: String,
    code: String,
}

#[derive(Debug, Deserialize)]
struct Pools {
    pool_a: PoolSections,
    pool_b: PoolSections,
}

#[derive(Debug, Deserialize)]
struct PoolSections {
    main: Vec<Card>,
    extra: Vec<Card>,
    side: Vec<Card>,
}

fn load_pools_from_file<P: AsRef<Path>>(path: P) -> Result<Pools, Box<dyn std::error::Error>> {
    let data = fs::read_to_string(path)?;
    let pools: Pools = serde_json::from_str(&data)?;
    Ok(pools)
}

fn pick_random_cards(cards: &[Card], count: usize) -> Vec<Card> {
    let mut rng = rand::thread_rng();
    let mut deck = Vec::new();

    let available_count = cards.len();
    if count >= available_count {
        deck.extend_from_slice(cards);
    } else {
        deck.extend(cards.choose_multiple(&mut rng, count).cloned());
    }

    deck
}

fn generate_deck(pools: &Pools, combine: bool, main_size: usize, extra_size: usize, side_size: usize) -> (Vec<Card>, Vec<Card>, Vec<Card>) {
    let main_source = if combine {
        [&pools.pool_a.main[..], &pools.pool_b.main[..]].concat()
    } else {
        pools.pool_a.main.clone()
    };

    let extra_source = if combine {
        [&pools.pool_a.extra[..], &pools.pool_b.extra[..]].concat()
    } else {
        pools.pool_a.extra.clone()
    };

    let side_source = if combine {
        [&pools.pool_a.side[..], &pools.pool_b.side[..]].concat()
    } else {
        pools.pool_a.side.clone()
    };

    let main_deck = pick_random_cards(&main_source, main_size);
    let extra_deck = pick_random_cards(&extra_source, extra_size);
    let side_deck = pick_random_cards(&side_source, side_size);

    (main_deck, extra_deck, side_deck)
}

/// Genera stringa YDK dal deck (main, extra, side)
fn generate_ydk(main: &[Card], extra: &[Card], side: &[Card]) -> String {
    let mut ydk = String::new();
    ydk.push_str("#created by deck-generator-rust\\n");
    ydk.push_str("#main\\n");
    for card in main {
        ydk.push_str(&format!("{}\\n", card.code));
    }
    ydk.push_str("#extra\\n");
    for card in extra {
        ydk.push_str(&format!("{}\\n", card.code));
    }
    ydk.push_str("#side\\n");
    for card in side {
        ydk.push_str(&format!("{}\\n", card.code));
    }
    ydk
}

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let args = Args::parse();

    let pools = load_pools_from_file(&args.pool_file)?;
    let (main_deck, extra_deck, side_deck) = generate_deck(&pools, args.combine, args.main_deck_size, args.extra_deck_size, args.side_deck_size);

    let ydk = generate_ydk(&main_deck, &extra_deck, &side_deck);
    let filename = if args.combine { "deck_combined.ydk" } else { "deck_a.ydk" };
    fs::write(filename, ydk)?;

    println!("Deck generato e salvato in {}", filename);

    Ok(())
}
'''

pools_extended_json = {
  "pool_a": {
    "main": [
      { "name": "Black Luster Soldier", "code": "72989439" },
      { "name": "Chaos Dragon Levianeer", "code": "39512984" },
      { "name": "Effect Veiler", "code": "97268402" }
    ],
    "extra": [
      { "name": "Knightmare Phoenix", "code": "40908371" },
      { "name": "Accesscode Talker", "code": "39765958" }
    ],
    "side": [
      { "name": "Ash Blossom & Joyous Spring", "code": "14558127" },
      { "name": "Droll & Lock Bird", "code": "94145021" }
    ]
  },
  "pool_b": {
    "main": [
      { "name": "Cyber Dragon", "code": "70095154" },
      { "name": "Jizukiru, the Star Destroying Kaiju", "code": "29804619" }
    ],
    "extra": [
      { "name": "Borreload Furious Dragon", "code": "24371632" },
      { "name": "Topologic Bomber Dragon", "code": "12644061" }
    ],
    "side": [
      { "name": "Dark Ruler No More", "code": "20932152" },
      { "name": "Evenly Matched", "code": "15693423" }
    ]
  }
}

readme_md = '''# Deck Generator Rust

Generatore casuale di deck per EdoPro, usa pool di carte definiti in JSON.

## Come usare

1. Clona il repo o scompatta lo zip:
