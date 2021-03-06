<template>
  <div class="d-flex flex-column ma-4">
    <span class="title">{{ $t("message.page_title") }}</span>
    <v-card class="ma-2">
      <v-card-title class="ma-1">{{
        $t("message.subject_title")
      }}</v-card-title>
      <v-card-text>{{ subject }} </v-card-text>
    </v-card>
    <v-card class="ma-2">
      <v-card-title>{{ $t("message.from_title") }}</v-card-title>
      <v-card-text>{{ from }}</v-card-text>
    </v-card>
    <v-card class="ma-2" v-if="notFound == true">
      <v-card-title class="ma-4">{{
        $t("message.not_found_title")
      }}</v-card-title>
    </v-card>
    <v-card class="ma-2" v-if="notFound == false">
      <v-card-title>{{ $t("message.judgement_title") }}</v-card-title>
      <v-card-text>{{ classificate }}</v-card-text>
    </v-card>
    <v-card class="ma-2" v-if="notFound == false">
      <v-card-title>{{ $t("message.category_score_title") }}</v-card-title>
      <v-list-item v-for="score in scores" :key="score.category">
        <v-list-item-content>
          <v-list-item-title
            class="ma-1"
            v-text="score.category"
          ></v-list-item-title>
          <v-list-item-subtitle class="ma-1"
            >{{ score.score }}
            {{ $t("message.unit_score") }}</v-list-item-subtitle
          >
        </v-list-item-content>
      </v-list-item>
    </v-card>
    <v-card v-for="category in wordscore" :key="category.word" class="ma-2">
      <v-card-title
        >{{ category.category }}
        {{ $t("message.top_score_word_title") }}</v-card-title
      >
      <v-list-item v-for="word in category.words" :key="word.word">
        <v-list-item-title>{{ word.word }}</v-list-item-title>
        <v-list-item-subtitle
          >{{ word.score }} {{ $t("message.unit_score") }}</v-list-item-subtitle
        >
        <v-list-item-subtitle
          >{{ word.count }} {{ $t("message.unit_count") }}</v-list-item-subtitle
        >
      </v-list-item>
    </v-card>
    <v-card class="ma-2" v-if="notFound == false">
      <v-card-title>{{ $t("message.target_text_title") }}</v-card-title>
      <div class="caption ma-2">{{ targetText }}</div>
    </v-card>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from "vue-property-decorator"
import LogEntry from "../../../models/LogEntry"
import MessageUtil from "../../../lib/MessageUtil"
import TagUtil from "../../../lib/TagUtil"
import Tag from "../../../models/Tag"
import TotalScore from "../../../models/TotalScore"

@Component
export default class App extends Vue {
  // コストラクタで非同期処理で変数初期化してもコンパイル通らないので仕方なくundefined無視
  private subject_: string = ""
  public get subject(): string {
    return this.subject_
  }

  private from_: string = ""
  public get from(): string {
    return this.from_
  }

  private logEntry_: LogEntry = new LogEntry()

  private targetText_: string = ""
  public get targetText(): string {
    return this.targetText_
  }

  private classificate_: string = ""
  public get classificate(): string {
    return this.classificate_
  }
  public set classificate(v: string) {
    this.classificate_ = v
  }

  private scores_: ScoresEachTag[] = []
  public get scores(): ScoresEachTag[] {
    return this.scores_
  }
  public set scores(v: ScoresEachTag[]) {
    this.scores_ = v
  }
  private wordscore_: WordEachCategory[] = []
  public get wordscore(): WordEachCategory[] {
    return this.wordscore_
  }

  private notFound_: boolean = true
  private get notFound(): boolean {
    return this.notFound_
  }

  private tags_: Tag[] = []

  private created() {
    this.Initialize()
    // タブがアクティブになったときに選択されたメール情報で再描画する為のフック
    browser.tabs.onActivated.addListener(this.refresh)
  }

  private async refresh(info: browser.tabs.activeInfo) {
    const thisTab = await browser.tabs.getCurrent()
    if (info.tabId == thisTab.id) {
      this.Initialize()
    }
  }

  private async Initialize() {
    const header = await this.getTargetMessage()
    this.subject_ = header.subject
    this.from_ = header.author

    this.notFound_ = false
    this.logEntry_.id = await MessageUtil.getMailMessageId(header)
    if ((await this.logEntry_.load()) == false) {
      this.notFound_ = true
      return
    }

    this.tags_ = await TagUtil.load()
    // 非同期で処理
    this.showClassifficateTag()
    this.showScore()
    this.showWordScore()
    this.targetText_ = this.logEntry_.targetText.join(" ")
  }

  /**
   * 表示用に単語 > カテゴリと記録されているログをカテゴリ > 単語の配列に変換
   * 単語の中で最もスコアの高いカテゴリに分類
   * さらにトップ10件に絞る
   */
  private async showWordScore(): Promise<void> {
    // ワードスコア初期化
    this.wordscore_ = []

    const scoreEachWord = this.logEntry_.scoreEachWord
    for (const word in scoreEachWord) {
      const scores = scoreEachWord[word].score
      const count = scoreEachWord[word].count
      let bestCategory = ""
      let bestScore = {} as {
        word: string
        count: number
        score: number
      }
      bestScore = {
        word: "",
        count: 0,
        score: 0,
      }
      // 単語毎カテゴリ毎のループ
      for (const category in scores) {
        const score = scores[category]
        // ワードの中で最もスコアの高いカテゴリを選択する
        if (bestScore.score <= score) {
          bestCategory = category
          bestScore = {
            word: word,
            count: count,
            score: score,
          }
        }
      }

      let catName = this.getTagName(bestCategory)
      if (catName == undefined) {
        // 消されたタグの場合は値が取れない可能性がある
        catName = "削除されたタグ"
      }

      // カテゴリ毎の単語一覧に追加
      if (bestScore.score != 0) {
        const ret = this.wordscore_.find((item) => item.category === catName)
        if (ret == undefined) {
          const ary = []
          ary.push(bestScore)
          this.wordscore_.push({
            category: catName,
            words: ary,
          })
        } else {
          ret.words.push(bestScore)
        }
      }
    }

    for (const category of this.wordscore_) {
      // ソート
      category.words.sort((a, b) => b.score - a.score)
      // トップ10件の単語のみに表示を削る
      category.words = category.words.slice(0, 9)
    }
  }

  private getTagName(key: string): string | undefined {
    const tag = this.tags_.find((item) => {
      return item.key === key
    })

    if (tag == undefined) return undefined
    return tag.name
  }

  private async showScore(): Promise<void> {
    //　スコア初期化
    this.scores_ = []

    for (const score of this.logEntry_.score) {
      const tag = this.getTagName(score.category)
      if (tag != undefined) {
        this.scores_.push({
          category: tag,
          score: score.score,
        })
      }
    }
  }

  private async showClassifficateTag(): Promise<void> {
    const tag = this.getTagName(this.logEntry_.classifiedTag)
    if (tag != undefined) {
      this.classificate_ = tag
    }
  }

  private async getTargetMessage(): Promise<browser.messages.MessageHeader> {
    const header = (await browser.storage.sync.get("logTarget")) as
      | undefined
      | {
          logTarget: browser.messages.MessageHeader
        }
    if (header == undefined)
      throw new Error("Not save MessageHeader in storage")
    return header.logTarget
  }
}

/**
 * スコア表示用のオブジェクト定義
 */
interface ScoresEachTag {
  category: string
  score: number
}

interface ScoreEachWord {
  word: string
  count: number
  score: number
}

interface WordEachCategory {
  category: string
  words: ScoreEachWord[]
}
</script>

<style lang="scss" scoped>
p {
  font-size: 20px;
}
</style>
