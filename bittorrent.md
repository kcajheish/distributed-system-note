Incentives Build Robustness in BitTorrent

purpose of peer2peer

- redistribute cost of upload to downloader
  - making hosting a file with unlimited downloader possible

difficulty

- overhead
  - determin peer/part of the file to download
- churn rate
  - disconnect after a few hours
- fairness
  - download rate = upload rate for each person

num of complete/incomplete downloader fall exponentially after a peak

.torrent file

- store file metadata, tracker ip

tracker knows below items

- store peer ip and send those ip to downloader
- gather download/upload statistics from downloader

seed

- downloader who has a complete copy of file

file is splitted into fixed size block(MB)

- generate hash for each block
- downloader report to the tracker and checks the downloaded block with the hash.
  - tracker know which peer has what part of the file

pipeline

- pend requests to reduce latency
  - around 5 requests
- each block is divided into subpiece(KB)

piece selection -> performance

- get subpieces in one piece first
- rarest first and most common last
- random first piece at the begining
  - download popular block so the downloader has piece to upload

endgame mode

choking algorithm

- tit for tat
  - cooperate -> upload
  - to not -> choke(stop uploading)
- keep every peer download rate high; lower peer's rate if peer doesn't upload

patero efficiency

- if both peers get poor return from upload, they can download from each other
- choose peer who gives best download rate
  - use rolling time window(20 sec)
  - long term rate is not used since bandwidth shifts rapidly
  - rate is calculated for every 10 seconds, the change takes effects in the next 10 seconds

optimistic unchoking

- for every 30 sec, unchoke a peer
- explore choked peer who may have better download rate

anti snubbing

upload only
