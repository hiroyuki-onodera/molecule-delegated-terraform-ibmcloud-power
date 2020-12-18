
https://qiita.com/hiroyuki_onodera/items/07d5d45fef3db71f3da8

# 課題:

Moleculeを使ってplaybookのテストなどを行う際に、クラウドにテスト環境を立ち上げる場合、terraformを使用すると以下のメリットがある。

- 様々なクラウドに対応可能
- 冪等性はterraformにて対応

ここでは IBM Cloud PowerSystems Virtual ServerのAIX環境を例に設定を試みる。


AWS EC2環境の例は以下の通り
https://qiita.com/hiroyuki_onodera/items/aef6194b5da425c198d8
terraformのtfファイル以外はほぼ同等


# 環境:

- macOS 10.15.7
- Python(pyenv) 3.8.2
- ansible 2.10.2
- molecule 3.1.5
- Terraform v0.13.4
- yq 2.10.1 (オプション)
- vm接続に使用するssh-key ~/.ssh/id_rsa, ~/.ssh/id_rsa.pub
- 使用する環境変数 IC_API_KEY, IC_REGION, TF_VAR_pi_cloud_instance_id


# 主要ファイル:

```
.
├── molecule
│   └── default
│       ├── tf_common.tf.yml          # terraform用AIX環境設定ファイル1
│       ├── tf_ins01.tf.yml           # terraform用AIX環境設定ファイル2
│       │
│       ├── (tf_common.tf.json)       # tf_common.tf.ymlから生成
│       ├── (tf_ins01.tf.json)        # tf_ins01.tf.ymlから生成
│       │
│       ├── (instance_conf-ins01.yml) # terraform作成
│       │                             # moleculeに連携するインスタンスへの接続情報
│       │
│       ├── (terraform.tfstate)       # terraform作成。ステータス情報
│       │
│       ├── molecule.yml              # molecule定義情報
│       ├── create.yml                # molecule vm作成playbook
│       ├── prepare.yml               # molecule vm事前準備playbook
│       ├── converge.yml              # molecule vmへの変更反映playbook
│       ├── verify.yml                # molecule vm検証playbook
│       └── destroy.yml               # molecule vm削除playbook
└── tasks
    └── main.yml
```

サンプルコードは以下
https://github.com/hiroyuki-onodera/molecule-delegated-terraform-ibmcloud-power

```
$ git clone https://github.com/hiroyuki-onodera/molecule-delegated-terraform-ibmcloud-power.git
or
$ git clone git@github.com:hiroyuki-onodera/molecule-delegated-terraform-ibmcloud-power.git
or
$ gh repo clone hiroyuki-onodera/molecule-delegated-terraform-ibmcloud-power
```


# CH1: Terraform による IBM Cloud AIX環境管理

## 事前準備:

IBM Cloudにアカウントを作成してログイン
https://cloud.ibm.com/login

https://qiita.com/c_u/items/10e3295023aa1a3b3c96
を参考に、以下の情報を入手。

- IBM Cloud API キー
- pi_cloud_instance_id
- リージョン情報

### IBM Cloud API キー

IBM Cloud コンソール => (上部バーの)管理 => (プルダウンメニューの)アクセス(IAM) => (左メニューの)API キー => 「IBM Cloud API キーの作成」ボタンから作成して使用
作成時に IBM Cloud API キーの確認やファイルとしてのダウンロードが可能

### Power Systems Virtual Serverサービスの作成

ダッシュボード右上のリソースの作成などからカタログメニューに行き、Power Systems Virtual Serverを検索して選択
リージョンの選択 にて、この例では 東京 04 を選択
右下の作成ボタンを押す

### pi_cloud_instance_id

上で作成したサービスを示すID。
作成後に確認するには、IBM Cloud Console => サービスリスト => Services下の、作成した Power Systems Virtual Server サービスの行を選択 => 右に表示されるサービス情報の GUID が該当。


## 入手した情報を環境変数に設定


bashの場合の例

```
$ cat <<EOC >> ~/.bash_profile
export IC_API_KEY="xxxxxxxxxxxxxxxxxxxxxxxx-xxxxxxxxxxxxxxxxxxx"
export IC_REGION=tok  # 東京の場合 tok, ダラスの場合 us-south
export TF_VAR_pi_cloud_instance_id=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
EOC
$ . ~/.bash_profile
```

特にIC_API_KEYはgithubやQiitaなどで公開しない様に注意が必要。

## terraform による providerやネットワーク定義



- .tfフォーマットは他のユーティリティとの連携などが困難
- .tf.jsonフォーマットは人が直接使用するには難しい
- ここでは、.tf.jsonをYAMLで表記したファイルにてPOWER環境設定を行い、terraform使用前にjson化する
- 最初から.tfフォーマットや.tf.jsonフォーマットで記述、管理するならばYAML管理は不要
- tf_common.tf.yml は、providerやネットワーク定義など個別のインスタンスに1:1で対応しない設定をまとめたもの。

```yaml:molecule/default/tf_common.tf.yml
---
terraform:
  required_version: ">= 0.13.3"
  required_providers:
    ibm:
      source: ibm-cloud/ibm
      version: 1.13.1
variable:
  pi_cloud_instance_id: {}
data:
  ibm_pi_image:
    aix_image:
      pi_image_name: 7200-04-01
      pi_cloud_instance_id: "${var.pi_cloud_instance_id}"
resource:
  ibm_pi_network:
    power_networks_public:
      pi_network_name: public_net
      pi_network_type: pub-vlan
      pi_dns:
      - 8.8.4.4
      pi_cloud_instance_id: "${var.pi_cloud_instance_id}"
```

この例では1つのインターネット接続ネットワークを作成して使用。

## terraform による インスタンス固有の定義

- tf_インスタンス名.tf.ymlは、インスタンスに1:1で対応する資源をまとめたもの
- インスタンス毎に1ファイルの想定
- j2などテンプレートエンジンを用いての作成は、どの範囲までユーザー指定とするかなど環境、プロジェクト依存の項目が多く、不必要に複雑化する為に断念。シンプルにこのファイル内での直接指定による設定としている。
- 作成したインスタンスへの接続情報はlocal_file:リソースを用いてMoleculeに連携する
- ユーザー利便を考慮してsshログイン方法をoutputにて表示

```yaml:molecule/default/tf_ins01.tf.yml
---
locals:
  ins_name-ins01: ins01
resource:
  ibm_pi_key:
    key-ins01:
      pi_key_name: key-ins01
      pi_ssh_key: ${file("~/.ssh/id_rsa.pub")}
      pi_cloud_instance_id: "${var.pi_cloud_instance_id}"
  ibm_pi_volume:
    ds_volume-ins01-01:
      pi_volume_name: tier3-10g-ins01-01
      pi_volume_type: tier3
      pi_volume_size: 10
      pi_volume_shareable: false
      pi_cloud_instance_id: "${var.pi_cloud_instance_id}"
  ibm_pi_instance:
    ins-ins01:
      pi_instance_name: "${local.ins_name-ins01}"
      pi_memory: 2
      pi_processors: 0.25
      pi_proc_type: shared
      pi_image_id: "${data.ibm_pi_image.aix_image.id}"
      pi_volume_ids:
      - "${ibm_pi_volume.ds_volume-ins01-01.volume_id}"
      pi_network_ids:
      - "${ibm_pi_network.power_networks_public.network_id}"
      pi_key_pair_name: "${ibm_pi_key.key-ins01.pi_key_name}"
      pi_sys_type: s922
      pi_replication_policy: none
      pi_replication_scheme: suffix
      pi_replicants: 1
      pi_cloud_instance_id: "${var.pi_cloud_instance_id}"
  local_file:
    instance_conf-ins01:
      filename: "./instance_conf-ins01.yml"
      content: |
        # this file is maintaind by terraform
        ---
        instance: ${local.ins_name-ins01}
        address: ${ibm_pi_instance.ins-ins01.addresses[0]["external_ip"]}
        user: root
        port: 22
        identity_file: ~/.ssh/id_rsa
output:
  ssh-ins01:
    value: >-
      ssh
      -o UserKnownHostsFile=/dev/null
      -o StrictHostKeyChecking=no
      -i ~/.ssh/id_rsa
      root@${ibm_pi_instance.ins-ins01.addresses[0]["external_ip"]}
```

この例では、自動的に割り振られるOS用のディスク以外に、追加で10GBのディスクを割り当て。

ちなみに、追加のディスクなしで、最小構成(IBM POWER9 s922, 0.25 コア, メモリー 2 GB, プロセッサー 上限なし共有)とした場合、割り振ったままだと月額6073円程度、Tier 3 10GBのディスクを追加して月額6199円程度の様子。
負担を減らすには都度削除しましょう。

local_fileを使用して、インスタンスへの接続方法をmoleculeに対して連携する

## terraformによるデプロイ、デストロイ確認

### tf.jsonファイル用意

最初から.tf.jsonや.tfファイルを使用しているならば不要
yqユーティリティによりYAMLファイルからterraform用の.tf.jsonファイルに変換

```
$ cd molecule/default/
$ yq . common.tf.yml > common.tf.json
$ yq . ins01.tf.yml > ins01.tf.json
```

yqはMACにデフォルトでは導入されていない。
以下の様にpythonなどでも代替可能。

```
$ cd molecule/default/
$ cat common.tf.yml | python -c "import yaml; import json; import sys; print(json.dumps(yaml.load(sys.stdin, Loader=yaml.FullLoader), indent=2))" > common.tf.json
$ cat ins01.tf.yml | python -c "import yaml; import json; import sys; print(json.dumps(yaml.load(sys.stdin, Loader=yaml.FullLoader), indent=2))" > ins01.tf.json
```


### terraform初期化 (terraform init)

```
$ terraform init

Initializing the backend...

Initializing provider plugins...
- Finding ibm-cloud/ibm versions matching "1.13.1"...
- Finding latest version of hashicorp/local...
- Installing ibm-cloud/ibm v1.13.1...
- Installed ibm-cloud/ibm v1.13.1 (unauthenticated)
- Installing hashicorp/local v1.4.0...
- Installed hashicorp/local v1.4.0 (unauthenticated)

...
Terraform has been successfully initialized!
...
```

プロバイダーファイルは./.terraform/plugins下に置かれる。
もし~/.terraform.d/pluginsに事前に配置しておくと、.terraform/plugins下はシンボリックリンクとなる。

### IBM Cloud AIX環境作成 (terraform apply)

```
$ terraform apply --auto-approve
data.ibm_pi_image.aix_image: Refreshing state...
ibm_pi_key.key-ins01: Creating...
ibm_pi_volume.ds_volume-ins01-01: Creating...
ibm_pi_network.power_networks_public: Creating...
ibm_pi_key.key-ins01: Creation complete after 2s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/key-ins01]
ibm_pi_volume.ds_volume-ins01-01: Still creating... [10s elapsed]
ibm_pi_network.power_networks_public: Still creating... [10s elapsed]
ibm_pi_volume.ds_volume-ins01-01: Creation complete after 16s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/8fd77549-b581-4d1b-a1b1-cb88faefb2de]
ibm_pi_network.power_networks_public: Creation complete after 17s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/4872551a-b9c4-4830-8d97-06e9d7ee9265]
ibm_pi_instance.ins-ins01: Creating...
ibm_pi_instance.ins-ins01: Still creating... [10s elapsed]
ibm_pi_instance.ins-ins01: Still creating... [20s elapsed]
...
ibm_pi_instance.ins-ins01: Still creating... [11m41s elapsed]
ibm_pi_instance.ins-ins01: Creation complete after 11m50s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/8239f3a9-508e-4d0b-9e8c-dd85db804934]
local_file.instance_conf-ins01: Creating...
local_file.instance_conf-ins01: Creation complete after 0s [id=658a4b3906b109fa6def47c68083784b4b0e9e6b]

Apply complete! Resources: 5 added, 0 changed, 0 destroyed.

Outputs:

ssh-ins01 = ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa root@128.168.100.115
```

### AIXインスタンスにsshログイン,ログアウト

```
$ ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa root@128.168.100.115
Warning: Permanently added '128.168.100.115' (RSA) to the list of known hosts.
*******************************************************************************
*                                                                             *
*                                                                             *
*  Welcome to AIX Version 7.2!                                                *
*                                                                             *
*                                                                             *
*  Please see the README file in /usr/lpp/bos for information pertinent to    *
*  this release of the AIX Operating System.                                  *
*                                                                             *
*                                                                             *
*******************************************************************************
# lspv
hdisk0          none                                None
hdisk1          00f6db0af58e9775                    rootvg          active
# ^D
Connection to 128.168.100.115 closed.
$
```

### terraformを用いて作成した接続情報の確認

```
$ cat instance_conf-ins01.yml
# this file is maintaind by terraform
---
instance: ins01
address: 128.168.100.115
user: root
port: 22
identity_file: ~/.ssh/id_rsa
```

### AIX資源削除 (terraform destroy)

```
$ terraform destroy --auto-approve
...

Destroy complete! Resources: 5 destroyed.
```

# CH2. Moleculeとの連携

以降はEC2対象時とほぼ変わらない。(ログを除く)


## Molecule設定ファイル (molecule.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるmolecule.ymlファイルを編集
platformsのnameをterraform設定ファイルと合わせておく

```yaml:molecule/default/molecule.yml
---
dependency:
  name: galaxy
driver:
  name: delegated
platforms: # terraformにて作成するインスタンスと名前が合致する必要がある
  - name: ins01
provisioner:
  name: ansible
verifier:
  name: ansible
```

## Molecule用テスト環境作成playbook (create.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるcreate.ymlファイルを編集
Dump taskは無変更

```yaml:molecule/default/create.yml
---
- name: Create
  hosts: localhost
  connection: local
  gather_facts: false
  #no_log: "{{ molecule_no_log }}"
  tasks:

  # molecule.ymlとterraform設定の整合性検証用1(オプション)
  - name: set_fact molecule_platforms
    set_fact:
      molecule_platforms: "{{ molecule_yml.platforms|map(attribute='name')|list }}"
  - debug:
      var: molecule_platforms

  # インスタンス作成処理本体(必須)
  - name: terraform apply -auto-approve
    shell: |-
      set -ex
      exec 2>&1
      which yq
      which terraform
      ! ls *.tf.json || rm *.tf.json
      ! ls instance_conf-*.yml || rm instance_conf-*.yml
      ls *.tf.yml | awk '{TO=$1; sub(/.tf.yml$/,".tf.json",TO); print "yq . "$1" > "TO}' | bash -x
      terraform init -no-color
      terraform apply -auto-approve -no-color || terraform apply -auto-approve -no-color
    register: r
  - debug:
      var: r.stdout_lines

  # Moleculeに連携する各インスタンスへの接続情報をlistにまとめる(必須)
  - name: Make instance_conf from terraform localfile
    vars:
      instance_conf: []
    with_fileglob:
    - instance_conf-*.yml
    set_fact:
      instance_conf: "{{ instance_conf + [lookup('file',item)|from_yaml] }}"

  # molecule.ymlとterraform設定の整合性検証用2(オプション)
  - name: set_fact terraform_platforms
    set_fact:
      terraform_platforms: "{{ instance_conf|map(attribute='instance')|list }}"
  - debug:
      var: terraform_platforms
  - name: Check molecule_platforms is included in terraform_platforms
    assert:
      that:
      - "{{ (molecule_platforms|difference(terraform_platforms)) == [] }}"

  # Moleculeに連携する各インスタンスへの接続情報をファイル出力(必須)
  - name: Dump instance config
    copy:
      content: |
        # Molecule managed

        {{ instance_conf | to_json | from_json | to_yaml }}
      dest: "{{ molecule_instance_config }}"
```
## Molecule用テスト環境準備playbook (prepare.yml)

ターゲットシステムにpythonが入っていなければ、rawモジュールを使用してpythonの導入をおこなったりすることに使用可能。
ここでは、接続ができるまで待たせる。
(terraform apply終了時点でインスタンスは上がっているので、ほとんど待たされないが。)

```yaml:molecule/default/prepare.yml
---
- name: Prepare
  hosts: all
  gather_facts: no
  tasks:
  - wait_for_connection:
```

## Molecule用テスト環境削除playbook (destroy.yml)

$ molecule init role ROLENAME --driver-name delegated
などにて作成されるdestroy.ymlファイルを編集。
Dump taskなどは無変更

```yaml:molecule/default/destroy.yml
---
- name: Destroy
  hosts: localhost
  connection: local
  gather_facts: false
  no_log: "{{ molecule_no_log }}"
  tasks:

  - name: Populate instance config
    shell: |-
      set -ex
      exec 2>&1
      terraform destroy --auto-approve -no-color || terraform destroy --auto-approve -no-color
    register: r
  - debug:
      var: r.stdout_lines

  - name: Populate instance config
    set_fact:
      instance_conf: {}

  - name: Dump instance config
    copy:
      content: |
        # Molecule managed

        {{ instance_conf | to_json | from_json | to_yaml }}
      dest: "{{ molecule_instance_config }}"
```


## テスト対象roleを呼び出すplaybook (converge.yml)

このroleのディレクトリー名変更に追随できる様に修正している。

```yaml:molecule/default/converge.yml
---
- name: Converge
  hosts: all
  tasks:
    - name: "Include {{ playbook_dir|dirname|dirname|basename }}"
      include_role:
        name: "{{ playbook_dir|dirname|dirname|basename }}"
```

## テスト対象roleのtasks/main.yml

ここでは対象インスタンスのOS種類情報などを取得表示させる

```
---
- debug:
    var: ansible_distribution
- debug:
    var: ansible_distribution_version
- shell: cat /etc/motd
  changed_when: false
  register: r
- debug:
    var: r.stdout_lines

```

shellモジュールが冪等性チェックに掛からない様に changed_when: false を設定


## molecule test 実行例

molecule testにて、インスタンス作成、反映、削除まで通しで実行させてみる
実行ディレクトリーはこのroleのディレクトリー直下。

```
$ time molecule test
--> Test matrix

└── default
    ├── dependency
    ├── lint
    ├── cleanup
    ├── destroy
    ├── syntax
    ├── create
    ├── prepare
    ├── converge
    ├── idempotence
    ├── side_effect
    ├── verify
    ├── cleanup
    └── destroy

--> Scenario: 'default'
--> Action: 'dependency'
Skipping, missing the requirements file.
Skipping, missing the requirements file.
--> Scenario: 'default'
--> Action: 'lint'
--> Lint is disabled.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Populate instance config] ************************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost]

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'syntax'

    playbook: /Users/aa220269/repo/repo-test/cookbooks/molecule-delegated-terraform-ibmcloud-power/molecule/default/converge.yml
--> Scenario: 'default'
--> Action: 'create'

    PLAY [Create] ******************************************************************

    TASK [set_fact molecule_platforms] *********************************************
    ok: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "molecule_platforms": [
            "ins01"
        ]
    }

    TASK [terraform apply -auto-approve] *******************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "r.stdout_lines": [
            "+ which yq",
            "/Users/aa220269/.pyenv/shims/yq",
            "+ which terraform",
            "/usr/local/bin/terraform",
            "+ ls tf_common.tf.json tf_ins01.tf.json",
            "tf_common.tf.json",
            "tf_ins01.tf.json",
            "+ rm tf_common.tf.json tf_ins01.tf.json",
            "+ ls 'instance_conf-*.yml'",
            "ls: instance_conf-*.yml: No such file or directory",
            "+ ls tf_common.tf.yml tf_ins01.tf.yml",
            "+ awk '{TO=$1; sub(/.tf.yml$/,\".tf.json\",TO); print \"yq . \"$1\" > \"TO}'",
            "+ bash -x",
            "+ yq . tf_common.tf.yml",
            "+ yq . tf_ins01.tf.yml",
            "+ terraform init -no-color",
            "",
            "Initializing the backend...",
            "",
            "Initializing provider plugins...",
            "- Using previously-installed ibm-cloud/ibm v1.13.1",
            "- Using previously-installed hashicorp/local v1.4.0",
            "",
            "The following providers do not have any version constraints in configuration,",
            "so the latest version was installed.",
            "",
            "To prevent automatic upgrades to new major versions that may contain breaking",
            "changes, we recommend adding version constraints in a required_providers block",
            "in your configuration, with the constraint strings suggested below.",
            "",
            "* hashicorp/local: version = \"~> 1.4.0\"",
            "",
            "Terraform has been successfully initialized!",
            "",
            "You may now begin working with Terraform. Try running \"terraform plan\" to see",
            "any changes that are required for your infrastructure. All Terraform commands",
            "should now work.",
            "",
            "If you ever set or change modules or backend configuration for Terraform,",
            "rerun this command to reinitialize your working directory. If you forget, other",
            "commands will detect it and remind you to do so if necessary.",
            "+ terraform apply -auto-approve -no-color",
            "data.ibm_pi_image.aix_image: Refreshing state...",
            "ibm_pi_volume.ds_volume-ins01-01: Creating...",
            "ibm_pi_network.power_networks_public: Creating...",
            "ibm_pi_key.key-ins01: Creating...",
            "ibm_pi_key.key-ins01: Creation complete after 8s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/key-ins01]",
            "ibm_pi_volume.ds_volume-ins01-01: Still creating... [10s elapsed]",
            "ibm_pi_network.power_networks_public: Still creating... [10s elapsed]",
            "ibm_pi_network.power_networks_public: Still creating... [20s elapsed]",
            "ibm_pi_volume.ds_volume-ins01-01: Still creating... [20s elapsed]",
            "ibm_pi_volume.ds_volume-ins01-01: Creation complete after 22s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/9a860d22-dc4f-4e02-a56b-738244cede60]",
            "ibm_pi_network.power_networks_public: Creation complete after 23s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/22e3925f-269f-42b2-a899-c5c98eca0fd4]",
            "ibm_pi_instance.ins-ins01: Creating...",
            "ibm_pi_instance.ins-ins01: Still creating... [10s elapsed]",
...
            "ibm_pi_instance.ins-ins01: Still creating... [13m50s elapsed]",
            "ibm_pi_instance.ins-ins01: Creation complete after 13m59s [id=b3932fd0-26e5-4985-bbdf-72819abb3be9/135305c8-8849-4377-896d-4a155ab5eadf]",
            "local_file.instance_conf-ins01: Creating...",
            "local_file.instance_conf-ins01: Creation complete after 0s [id=93a71d7887ff289e218b0176765144ad26cd9b60]",
            "",
            "Apply complete! Resources: 5 added, 0 changed, 0 destroyed.",
            "",
            "Outputs:",
            "",
            "ssh-ins01 = ssh -o UserKnownHostsFile=/dev/null -o StrictHostKeyChecking=no -i ~/.ssh/id_rsa root@128.168.100.122"
        ]
    }

    TASK [Make instance_conf from terraform localfile] *****************************
    ok: [localhost] => (item=/Users/aa220269/repo/repo-test/cookbooks/molecule-delegated-terraform-ibmcloud-power/molecule/default/instance_conf-ins01.yml)

    TASK [set_fact terraform_platforms] ********************************************
    ok: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost] => {
        "terraform_platforms": [
            "ins01"
        ]
    }

    TASK [Check molecule_platforms is included in terraform_platforms] *************
    ok: [localhost] => {
        "changed": false,
        "msg": "All assertions passed"
    }

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=9    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'prepare'

    PLAY [Prepare] *****************************************************************

    TASK [wait_for_connection] *****************************************************
    ok: [ins01]

    PLAY RECAP *********************************************************************
    ins01                      : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'converge'

    PLAY [Converge] ****************************************************************

    TASK [Gathering Facts] *********************************************************
[WARNING]: Platform aix on host ins01 is using the discovered Python
interpreter at /usr/bin/python, but future installation of another Python
interpreter could change the meaning of that path. See https://docs.ansible.com
/ansible/2.10/reference_appendices/interpreter_discovery.html for more
information.
    ok: [ins01]

    TASK [Include molecule-delegated-terraform-ibmcloud-power] *********************

    TASK [molecule-delegated-terraform-ibmcloud-power : debug] *********************
    ok: [ins01] => {
        "ansible_distribution": "AIX"
    }

    TASK [molecule-delegated-terraform-ibmcloud-power : debug] *********************
    ok: [ins01] => {
        "ansible_distribution_version": "7.2"
    }

    TASK [molecule-delegated-terraform-ibmcloud-power : shell] *********************
    ok: [ins01]

    TASK [molecule-delegated-terraform-ibmcloud-power : debug] *********************
    ok: [ins01] => {
        "r.stdout_lines": [
            "*******************************************************************************",
            "*                                                                             *",
            "*                                                                             *",
            "*  Welcome to AIX Version 7.2!                                                *",
            "*                                                                             *",
            "*                                                                             *",
            "*  Please see the README file in /usr/lpp/bos for information pertinent to    *",
            "*  this release of the AIX Operating System.                                  *",
            "*                                                                             *",
            "*                                                                             *",
            "*******************************************************************************"
        ]
    }

    PLAY RECAP *********************************************************************
    ins01                      : ok=5    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Scenario: 'default'
--> Action: 'idempotence'
Idempotence completed successfully.
--> Scenario: 'default'
--> Action: 'side_effect'
Skipping, side effect playbook not configured.
--> Scenario: 'default'
--> Action: 'verify'
--> Running Ansible Verifier

    PLAY [Verify] ******************************************************************

    TASK [Example assertion] *******************************************************
    ok: [ins01] => {
        "changed": false,
        "msg": "All assertions passed"
    }

    PLAY RECAP *********************************************************************
    ins01                      : ok=1    changed=0    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

Verifier completed successfully.
--> Scenario: 'default'
--> Action: 'cleanup'
Skipping, cleanup playbook not configured.
--> Scenario: 'default'
--> Action: 'destroy'

    PLAY [Destroy] *****************************************************************

    TASK [Populate instance config] ************************************************
    changed: [localhost]

    TASK [debug] *******************************************************************
    ok: [localhost]

    TASK [Populate instance config] ************************************************
    ok: [localhost]

    TASK [Dump instance config] ****************************************************
    changed: [localhost]

    PLAY RECAP *********************************************************************
    localhost                  : ok=4    changed=2    unreachable=0    failed=0    skipped=0    rescued=0    ignored=0

--> Pruning extra files from scenario ephemeral directory

real    17m8.125s
user    3m14.203s
sys     0m42.144s
```

意図通り、molecule testコマンドにより、AIX環境が作成され、playbookが適用され、最後に削除されている。


# 参考

Local PC から Terraform で IBM Power Systems Virtual Server on IBM Cloud 上にAIXサーバー をデプロイする
https://qiita.com/c_u/items/10e3295023aa1a3b3c96?utm_campaign=email&utm_content=link&utm_medium=email&utm_source=public_patch_accepted
