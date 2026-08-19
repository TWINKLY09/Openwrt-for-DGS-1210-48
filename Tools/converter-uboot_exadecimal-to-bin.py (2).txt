#!/usr/bin/env python3
"""
uboot_mdb_to_bin.py
====================
Reconstruit un fichier binaire depuis la sortie brute d'un `md.b` U-Boot
capturée via UART (minicom, picocom, screen, etc.)

Format attendu (U-Boot 1.1.x / 2013) :
  b8000000: 00 01 02 03 04 05 06 07 08 09 0a 0b 0c 0d 0e 0f    ................
  b8000010: 10 11 12 13 ...

Usage :
  python3 uboot_mdb_to_bin.py capture.txt flash_dump.bin

Options :
  --start-addr  Adresse de début attendue (hex, ex: b8000000) — pour validation
  --skip-errors Ignore les lignes mal formées au lieu de s'arrêter
  --verbose     Affiche la progression toutes les 256KB
"""

import sys
import re
import argparse
from pathlib import Path


# ── Patterns de lignes md.b ──────────────────────────────────────────────────
# Format standard U-Boot :  "b8000000: 00 01 02 ... 0f    ................"
# Variante sans ASCII    :  "b8000000: 00 01 02 ... 0f"
# Variante avec séparateur pipe : "b8000000: 00 01 ... | ................"
PATTERN = re.compile(
    r'^([0-9a-fA-F]{8})\s*:\s+'        # adresse 32 bits
    r'((?:[0-9a-fA-F]{2}\s+){1,16})'  # 1 à 16 octets hex
    r'.*$'                              # reste ignoré (ASCII, pipes...)
)

# Lignes à ignorer silencieusement
SKIP_PATTERNS = [
    re.compile(r'^music>\s*'),           # prompt U-Boot
    re.compile(r'^md\.b\s+'),            # écho de la commande
    re.compile(r'^\s*$'),               # ligne vide
    re.compile(r'^--More--'),           # pagination
    re.compile(r'^\x1b\['),             # séquences ANSI
]


def is_ignorable(line: str) -> bool:
    return any(p.match(line) for p in SKIP_PATTERNS)


def parse_capture(infile: Path, start_addr: int | None, skip_errors: bool, verbose: bool) -> bytes:
    data = bytearray()
    expected_addr = start_addr
    errors = 0
    lines_parsed = 0

    with open(infile, 'r', errors='replace') as f:
        for lineno, raw in enumerate(f, 1):
            line = raw.rstrip('\r\n')

            if is_ignorable(line):
                continue

            m = PATTERN.match(line)
            if not m:
                if skip_errors:
                    errors += 1
                    continue
                else:
                    print(f"[ERREUR] Ligne {lineno} non reconnue : {line!r}", file=sys.stderr)
                    print("  → utilise --skip-errors pour ignorer", file=sys.stderr)
                    sys.exit(1)

            addr = int(m.group(1), 16)
            hex_bytes = [int(b, 16) for b in m.group(2).split()]

            # Vérification de continuité
            if expected_addr is not None and addr != expected_addr:
                gap = addr - expected_addr
                if gap > 0:
                    print(f"[WARN] Ligne {lineno}: gap de {gap:#x} octets "
                          f"(attendu {expected_addr:#010x}, reçu {addr:#010x}) → rembourrage avec 0xFF",
                          file=sys.stderr)
                    data.extend(b'\xff' * gap)
                elif gap < 0:
                    print(f"[WARN] Ligne {lineno}: adresse en recul de {-gap:#x} octets — overlap ignoré",
                          file=sys.stderr)

            data.extend(hex_bytes)
            expected_addr = addr + len(hex_bytes)
            lines_parsed += 1

            if verbose and len(data) % (256 * 1024) < 16:
                print(f"  {len(data) / 1024:.0f} KB traités...", file=sys.stderr)

    if errors:
        print(f"[INFO] {errors} lignes ignorées (--skip-errors actif)", file=sys.stderr)

    print(f"[INFO] {lines_parsed} lignes parsées, {len(data)} octets ({len(data)/1024:.1f} KB)", file=sys.stderr)
    return bytes(data)


def main():
    parser = argparse.ArgumentParser(description="Reconstruit un binaire depuis un dump md.b U-Boot")
    parser.add_argument("input",  type=Path, help="Fichier texte capturé (sortie UART)")
    parser.add_argument("output", type=Path, help="Fichier binaire de sortie")
    parser.add_argument("--start-addr",  type=lambda x: int(x, 16), default=None,
                        help="Adresse de début attendue en hex (ex: b8000000)")
    parser.add_argument("--skip-errors", action="store_true",
                        help="Ignore les lignes mal formées")
    parser.add_argument("--verbose",     action="store_true",
                        help="Affiche la progression")
    args = parser.parse_args()

    if not args.input.exists():
        print(f"[ERREUR] Fichier introuvable : {args.input}", file=sys.stderr)
        sys.exit(1)

    print(f"[INFO] Lecture de {args.input} ...", file=sys.stderr)
    data = parse_capture(args.input, args.start_addr, args.skip_errors, args.verbose)

    args.output.write_bytes(data)
    print(f"[OK] Binaire écrit : {args.output} ({len(data)} octets / {len(data)/1024/1024:.2f} MB)")

    # Sanity checks basiques
    if len(data) == 0x1000000:
        print("[OK] Taille exacte 16MB — dump complet ✓")
    elif len(data) < 0x1000000:
        print(f"[WARN] Taille inférieure à 16MB — dump peut-être incomplet "
              f"(manque {0x1000000 - len(data):#x} octets)")
    else:
        print(f"[WARN] Taille supérieure à 16MB — vérifier le début/fin de capture")

    # Vérification que le début ressemble à de l'U-Boot (magic ou instructions MIPS)
    if data[:4] != b'\xff\xff\xff\xff':
        print(f"[INFO] Premiers octets : {data[:8].hex(' ')} — semble non-vide ✓")
    else:
        print("[WARN] Les 4 premiers octets sont 0xFF — peut indiquer un offset incorrect")


if __name__ == "__main__":
    main()
