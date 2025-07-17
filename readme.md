#score_wallet:This script loads wallet transaction data from a JSON file named user-wallet-transactions.json and checks if the top-level structure is a list. If so, it prints the first transaction entry; otherwise, it notifies that the data format is unexpected.

'import json

file_path = "user-wallet-transactions.json"

with open(file_path, "r") as file:
    data = json.load(file)

# Print top-level type
print(f"Top-level type: {type(data)}")

# If it's a list, print first transaction
if isinstance(data, list) and len(data) > 0:
    print("\nFirst transaction entry:")
    print(json.dumps(data[0], indent=2))
else:
    print("Unexpected data format")'





#extract_wallet_features:This script processes Aave V2 DeFi transaction data from multiple wallets, groups them by wallet address, and extracts meaningful behavioral features (like how often a wallet deposits, borrows, repays, etc.). The extracted features are then saved to a file called wallet_features.json.



import json
from collections import defaultdict
from datetime import datetime

# Load data
with open("user-wallet-transactions.json", "r") as file:
    transactions = json.load(file)

# Group transactions by wallet
wallets = defaultdict(list)
for tx in transactions:
    wallets[tx["userWallet"]].append(tx)

def extract_features(transactions):
    features = {
        "num_transactions": len(transactions),
        "num_deposits": 0,
        "num_borrows": 0,
        "num_repays": 0,
        "num_redeems": 0,
        "num_liquidations": 0,
        "total_deposit_usd": 0.0,
        "total_borrow_usd": 0.0,
        "total_repay_usd": 0.0,
        "total_redeem_usd": 0.0,
        "assets": set(),
        "active_days": set()
    }

    for tx in transactions:
        action = tx.get("action")
        action_data = tx.get("actionData", {})
        usd_price = float(action_data.get("assetPriceUSD", 0))
        amount = float(action_data.get("amount", 0))
        asset = action_data.get("assetSymbol")

        if asset:
            features["assets"].add(asset)

        # Handle timestamp safely
        try:
            date = datetime.utcfromtimestamp(tx["timestamp"]).date()
            features["active_days"].add(date)
        except Exception as e:
            print(f" Error with timestamp: {tx.get('timestamp')} - {e}")

        if action == "deposit":
            features["num_deposits"] += 1
            features["total_deposit_usd"] += usd_price * amount / 1e6

        elif action == "borrow":
            features["num_borrows"] += 1
            features["total_borrow_usd"] += usd_price * amount / 1e18

        elif action == "repay":
            features["num_repays"] += 1
            features["total_repay_usd"] += usd_price * amount / 1e18

        elif action == "redeemunderlying":
            features["num_redeems"] += 1
            features["total_redeem_usd"] += usd_price * amount / 1e6

        elif action == "liquidationcall":
            features["num_liquidations"] += 1

    return {
        "num_transactions": features["num_transactions"],
        "num_deposits": features["num_deposits"],
        "num_borrows": features["num_borrows"],
        "num_repays": features["num_repays"],
        "num_redeems": features["num_redeems"],
        "num_liquidations": features["num_liquidations"],
        "total_deposit_usd": round(features["total_deposit_usd"], 2),
        "total_borrow_usd": round(features["total_borrow_usd"], 2),
        "total_repay_usd": round(features["total_repay_usd"], 2),
        "total_redeem_usd": round(features["total_redeem_usd"], 2),
        "asset_count": len(features["assets"]),
        "active_days": len(features["active_days"])
    }

# Build the feature matrix
wallet_features = {}

print(" Extracting wallet features...\n")
for i, (wallet, txs) in enumerate(wallets.items(), 1):
    wallet_features[wallet] = extract_features(txs)
    print(f"Wallet {i}: {wallet}")
    print(json.dumps(wallet_features[wallet], indent=2))
    print("-" * 60)

# Save to file
with open("wallet_features.json", "w") as outfile:
    json.dump(wallet_features, outfile, indent=2)

print("\n Feature extraction complete. Saved to wallet_features.json")




#credit_scores:This script loads the behavioral features extracted for each wallet from wallet_features.json, calculates a creditworthiness score (0–1000 scale) for each wallet based on their activity, and saves the results to wallet_scores.json.
import json

# Load extracted wallet features
with open("wallet_features.json", "r") as file:
    wallet_features = json.load(file)

wallet_scores = {}

# Define weights (you can fine-tune these)
WEIGHTS = {
    "deposit": 0.4,
    "repay": 0.3,
    "redeem": 0.1,
    "liquidation_penalty": 0.2,
    "asset_bonus": 0.05,
    "activity_bonus": 0.05
}

def calculate_score(features):
    # Safely extract values with defaults
    deposit_usd = float(features.get("total_deposit_usd", 0))
    repay_usd = float(features.get("total_repay_usd", 0))
    redeem_usd = float(features.get("total_redeem_usd", 0))
    liquidations = int(features.get("num_liquidations", 0))
    asset_count = int(features.get("asset_count", 0))
    active_days = int(features.get("active_days", 0))

    # Normalize scores (scale and clamp between 0–1)
    deposit_score = min(deposit_usd / 1000, 1.0)
    repay_score = min(repay_usd / 1000, 1.0)
    redeem_score = min(redeem_usd / 500, 1.0)

    liquidation_penalty = min(liquidations * 0.1, 1.0)  # 10% penalty per liquidation
    asset_score = min(asset_count / 5, 1.0)
    activity_score = min(active_days / 30, 1.0)

    # Compute raw score (before penalty)
    base_score = (
        WEIGHTS["deposit"] * deposit_score +
        WEIGHTS["repay"] * repay_score +
        WEIGHTS["redeem"] * redeem_score +
        WEIGHTS["asset_bonus"] * asset_score +
        WEIGHTS["activity_bonus"] * activity_score
    ) * 1000

    # Apply liquidation penalty
    final_score = base_score * (1 - liquidation_penalty)

    return round(final_score, 2)

# Calculate scores
for wallet, features in wallet_features.items():
    try:
        score = calculate_score(features)
        wallet_scores[wallet] = score

        #  Add this line to print each wallet and its score
        print(f"Wallet: {wallet} → Score: {score}")

    except Exception as e:
        print(f" Error scoring wallet {wallet}: {e}")
        
# Save scores
with open("wallet_scores.json", "w") as outfile:
    json.dump(wallet_scores, outfile, indent=2)

print(" Credit scores calculated and saved to wallet_scores.json")




#generate_graph: I have generate the generate the graph on the basic of the wallet_scores:
import json
import matplotlib.pyplot as plt

# Load scores
with open("wallet_scores.json", "r") as f:
    wallet_scores = json.load(f)

# Score buckets (0–100, 100–200, ..., 900–1000)
buckets = [0] * 10
for score in wallet_scores.values():
    index = min(score // 100, 9)  # Ensure 1000 goes into last bucket
    buckets[int(index)] += 1

# Labels and values
labels = [f"{i*100}–{(i+1)*100}" for i in range(10)]

# Plot
plt.figure(figsize=(10, 6))
plt.bar(labels, buckets, color="skyblue", edgecolor="black")
plt.title("Wallet Score Distribution")
plt.xlabel("Score Range")
plt.ylabel("Number of Wallets")
plt.grid(axis='y', linestyle="--", alpha=0.7)
plt.tight_layout()
plt.savefig("score_distribution.png")
plt.show()