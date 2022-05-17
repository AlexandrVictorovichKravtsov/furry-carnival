# furry-carnival
akz
#!/usr/bin/php
<?php
namespace

class Application
{
    public $opts;

    public $mode;

    public $sourceFile;

    public $signatureFile;

    public $resultFile;

    public function run()
    {
        $this->opts = getopt("s:r:e:d:h");

        $this->getMode();

        if ($this->mode == "encode") {
            $this->getSourceFile();
            $this->getResultFile();
            $this->getSignatureFile(false);
            $this->encodeFile();
        } elseif ($this->mode == "decode") {
            $this->getSourceFile();
            $this->getResultFile();
            $this->getSignatureFile(true);
            $this->decodeFile();
        } elseif ($this->mode == "help") {
            $this->showHelp();
        } else {
            $this->showHelp();
            $this->showError("Invalid mode");
            exit(1);
        }
    }

    public function encodeFile()
    {
        $code = file_get_contents($this->sourceFile);
        printf(" > Source => %s:\n%s", $this->sourceFile, $code);

        printf(" > Signature => %s:\n", $this->signatureFile ? $this->signatureFile : "<none>");
        if ($this->signatureFile) {
            $signature = file_get_contents($this->signatureFile);
            $this->printBinaryCode($signature);
        } else {
            $signature = null;
        }

        $result = $this->encodeVersion2License($code, $signature);
        printf(" > Result => %s:\n", $this->resultFile ? $this->resultFile : "<none>");
        $this->printCode($result);
        if ($this->resultFile) {
            file_put_contents($this->resultFile, $result);
        }
    }

    public function decodeFile()
    {
        $code = file_get_contents($this->sourceFile);
        $code = $this->stripSpaces($code);
        printf(" > Source => %s:\n", $this->sourceFile);
        $this->printCode($code);

        $result = $this->decodeVersion2License($code, $signature);

        printf(" > Signature => %s:\n", $this->signatureFile ? $this->signatureFile : "<none>");
        $this->printBinaryCode($signature);
        if ($this->signatureFile) {
            file_put_contents($this->signatureFile, $signature);
        }
        printf(" > Result => %s:\n%s", $this->resultFile ? $this->resultFile : "<none>", $result);
        if ($this->resultFile) {
            file_put_contents($this->resultFile, $result);
        }
    }
    public function getSourceFile()
    {
        $opt = $this->mode == "encode" ? "e" : "d";
        if (!array_key_exists($opt, $this->opts)) {
            printf("ERROR: You need to specify source file\n");
            exit(1);
        }
        $this->sourceFile = $this->opts[$opt];
        if (!file_exists($this->sourceFile)) {
            printf("ERROR: Unable to find source file: %s\n", $this->sourceFile);
            exit(1);
      
    public function getResultFile()
    {
        if (!array_key_exists("r", $this->opts)) {
            $this->resultFile = false;
        } else {
            $this->resultFile = $this->opts["r"];
        }
    }

    public function getSignatureFile($optional)
    {
        if (!array_key_exists("s", $this->opts)) {
            return false;
        }
        $this->signatureFile = $this->opts["s"];
        if (!$optional && !file_exists($this->signatureFile)) {
            printf("ERROR: Unable to find signature file: %s\n", $this->signatureFile);
            exit(1);
        }
    }

    public function showError($message)
    {
        printf("ERROR: %s\n", $message);
    }

    public function showHelp()
    {
        printf("Atlassian Jira Keygen v2\n");
        printf("(jira will accept keys generated by this keygen only if patched for that)\n");
        printf("Usage: %s <-e|-d|-h> -f <source file> [-s <signature file>] [-r <result file>]\n", basename($_SERVER["argv"][0]));
        printf("Options:\n");
        printf(" -h    this screen\n");
        printf(" -e    encode license file and attach signature\n");
        printf(" -d    decode license file and detach signature\n");
        printf(" -s    signature file\n");
        printf(" -r    put results in file\n");
    }

    public function getMode()
    {
        if (array_key_exists("h", $this->opts)) {
            $this->mode = "help";
        } elseif (array_key_exists("e", $this->opts)) {
            $this->mode = "encode";
        } elseif (array_key_exists("d", $this->opts)) {
            $this->mode = "decode";
        } else {
            $this->mode = false;
        }
    }

    private function stripSpaces($code)
    {
        return str_replace(["\r", "\n", "\t", " "], "", $code);
    }

    private function printBinaryCode($text)
    {
        for ($i = 0; $i < strlen($text); $i++) {
            printf("%02X", ord(substr($text, $i, 1)));
            if (($i + 1) % 40 == 0) {
                printf("\n");
            }
        }
        printf("\n");
    }

    private function printCode($text)
    {
        for ($i = 0; $i < strlen($text); $i++) {
            printf("%s", substr($text, $i, 1));
            if (($i + 1) % 80 == 0) {
                printf("\n");
            }
        }
        printf("\n");
    }

    private function decodeVersion2License($code, &$signature = null)
    {
        $code = str_replace(["\r", "\n", "\t", " "], "", $code);

        $xPos = strrpos($code, "X");
        $ver = substr($code, $xPos + 1, 2);
        if ($ver != "02") {
            $this->showError("Invalid license version: $ver");
            exit(1);
        }
        //$code = substr($code, 0, intval($xPos + 3, 31));

        $code = substr($code, 0, intval($xPos, 31));

        printf(" > data:\n");
        $this->printCode($code);

        $code = base64_decode($code);

        printf(" > binary data:\n");
        $this->printBinaryCode($code);

        //Parse get data -> get size
        $sizeRec = unpack("N", substr($code, 0, 4));
        $size = $sizeRec[1];
        print("> size: \n ");
        print($sizeRec[1]);

        //Bypass 4 position (size) -> get text
        $text = substr($code, 4, $size);

        //Bypass $size + 4 -> get signature
        $signature = substr($code, $size + 4);

        //Check version of text
        $id = substr($text, 0, 5);
        $id2 = chr(13) . chr(14) . chr(12) . chr(10) . chr(15);
        if ($id != $id2) {
            $this->showError("Invalid license v2 format");
            exit(1);
        }
        //Get text after strip id (version)
        $text = substr($text, 5);   // strip id
        printf(" > zlib prefix:\n");
        $this->printBinaryCode(substr($text, 0, 2));

        //Get text after strip rfc1950 header
        $text = substr($text, 2);   // strip rfc1950 header
        printf(" > zlib suffix:\n");
        $this->printBinaryCode(substr($text, -4));

        $text = substr($text,0, strlen($text) - 4); //binhnt

        $text = gzinflate($text);   // decompress

        return $text;
    }

    private function encodeVersion2License($text, $signature = null)
    {
        $id = chr(13) . chr(14) . chr(12) . chr(10) . chr(15);
        $gzPrefix = "\x78\xDA";
        $gzSuffix = mhash(MHASH_ADLER32, $text);
        printf(" > zlib prefix:\n");
        $this->printBinaryCode($gzPrefix);
        printf(" > zlib suffix:\n");
        $this->printBinaryCode($gzSuffix);

        $text = $id . $gzPrefix . gzdeflate($text) . $gzSuffix;
        printf(" > size:\n");
        print(strlen($text));
        $size = pack("N", strlen($text));

        $data = $size .$text. $signature;

        printf(" > binary data:\n");
        $this->printBinaryCode($data);

        $data = trim(base64_encode($data));
        printf(" > data:\n");
        $this->printCode($data);

        $result = $data . "X" . "0" . "2" . base_convert(strlen($data), 10, 31);
        return $result;
    }
}

$app = new Application();
$app->run();
© 2022 GitHub, Inc.
Terms
Privacy
Security
Status
Docs
Contact GitHub
Pricing
API
Training
Blog
About
