<template>
  <div class="search-torrent">
    <v-alert :value="true" type="info" style="padding:8px 16px;">
      {{ words.title }} [{{ key }}], {{searchMsg}} {{skipSites}}
      <v-btn
        flat
        icon
        small
        color="white"
        @click.stop="doSearch"
        :title="words.reSearch"
        v-if="!loading && key!=''"
      >
        <v-icon>cached</v-icon>
      </v-btn>

      <!-- 无结果的站点 -->
      <v-btn
        v-if="searchResult.noResultsSites.length>0"
        class="mt-1"
        flat
        small
        color="white"
        @click.stop="showNoResultsSites=!showNoResultsSites"
      >
        <v-icon small class="mr-1" color="grey darken-2">face</v-icon>
        {{ words.noResultsSites }}
        {{ searchResult.noResultsSites.length }}
      </v-btn>

      <!-- 失败的站点 -->
      <v-btn
        v-if="searchResult.failedSites.length>0"
        class="mt-1"
        flat
        small
        color="white"
        @click.stop="showFailedSites=!showFailedSites"
      >
        <v-icon small class="mr-1" color="orange">warning</v-icon>
        {{ words.failedSites }}
        {{ searchResult.failedSites.length }}
      </v-btn>

      <v-btn
        v-if="searchResult.failedSites.length>0 && showFailedSites"
        class="mt-1"
        flat
        small
        color="white"
        @click.stop="reSearchFailedSites"
      >
        <v-icon small class="mr-1">autorenew</v-icon>
        {{ words.reSearchFailedSites }}
      </v-btn>
    </v-alert>
    <!-- 搜索队列-->
    <v-list small v-if="searchQueue && searchQueue.length">
      <template v-for="(item, index) in searchQueue">
        <v-list-tile :key="item.site.host">
          <v-list-tile-action>
            <v-progress-circular :size="18" :width="2" indeterminate color="primary"></v-progress-circular>
          </v-list-tile-action>

          <v-list-tile-content>
            <v-list-tile-title>
              <v-avatar size="18" class="mr-2">
                <img :src="item.site.icon">
              </v-avatar>
              {{item.site.name}} {{ words.searching }}
            </v-list-tile-title>
          </v-list-tile-content>

          <v-list-tile-action class="mr-5">
            <v-icon @click="abortSearch(item.site)" color="red" :title="words.cancelSearch">cancel</v-icon>
          </v-list-tile-action>
        </v-list-tile>
        <v-divider v-if="index + 1 < searchQueue.length" :key="'line'+item.site.host+index"></v-divider>
      </template>
    </v-list>
    <!-- 搜索结果列表 -->
    <v-card>
      <v-card-title>
        <v-flex xs6>
          <!-- 站点返回的搜索结果 -->
          <div v-if="searchSiteCount>1">
            <template v-for="(item, key) in searchResult.sites">
              <v-chip
                :key="key"
                label
                :color="item.length?'blue-grey darken-2':'grey'"
                text-color="white"
                small
                class="mr-2 py-3 pl-1"
                @click.stop="resetDatas(item)"
                :disabled="!item.length"
              >
                <v-icon class="mr-1" left v-if="key===words.allSites">public</v-icon>
                <template v-else>
                  <v-avatar class="mr-1" v-if="item.length>0">
                    <img :src="item[0].site.icon" style="width:60%;height:60%;">
                  </v-avatar>
                  <v-avatar class="mr-1" v-else>
                    <img :src="item.site.icon" style="width:60%;height:60%;">
                  </v-avatar>
                </template>
                <span>{{ key }}</span>
                <v-chip
                  label
                  :color="item.length?'blue-grey':'grey'"
                  small
                  text-color="white"
                  style="margin-right:-13px;"
                  class="ml-2 py-3"
                  disabled
                >
                  <span>{{ item.length || item.msg }}</span>
                </v-chip>
              </v-chip>
            </template>
          </div>

          <!-- 无结果的站点 -->
          <div v-if="searchResult.noResultsSites.length>0 && showNoResultsSites">
            <template v-for="(item, index) in searchResult.noResultsSites">
              <v-chip
                :key="index"
                label
                color="grey darken-1"
                text-color="white"
                small
                class="mr-2 py-3 pl-1"
                disabled
              >
                <template>
                  <v-avatar class="mr-1">
                    <img :src="item.site.icon" style="width:60%;height:60%;">
                  </v-avatar>
                </template>
                <span>{{ item.site.name }}</span>
                <v-chip
                  label
                  color="grey"
                  small
                  text-color="white"
                  style="margin-right:-13px;"
                  class="ml-2 py-3"
                  disabled
                >
                  <span>{{ item.msg }}</span>
                </v-chip>
              </v-chip>
            </template>
          </div>

          <!-- 站点返回的失败的站点 -->
          <div v-if="searchResult.failedSites.length>0 && showFailedSites">
            <template v-for="(item, index) in searchResult.failedSites">
              <v-chip
                :key="index"
                label
                color="orange darken-3"
                text-color="white"
                small
                class="mr-2 py-3 pl-1"
                disabled
              >
                <template>
                  <v-avatar class="mr-1">
                    <img :src="item.site.icon" style="width:60%;height:60%;">
                  </v-avatar>
                </template>
                <span>{{ item.site.name }}</span>
                <v-chip
                  label
                  :color="item.color"
                  small
                  text-color="white"
                  style="margin-right:-13px;"
                  class="ml-2 py-3"
                  disabled
                >
                  <a
                    v-if="item.url"
                    :href="item.url"
                    rel="noopener noreferrer nofollow"
                    target="_blank"
                  >{{ item.msg }}</a>
                  <span v-if="!item.url">{{ item.msg }}</span>
                </v-chip>
              </v-chip>
            </template>
          </div>

          <!-- 标签列表 -->
          <div class="mt-1">
            <template v-for="(item, key) in searchResult.tags">
              <v-chip
                :key="key"
                label
                :color="item.tag.color"
                text-color="white"
                small
                class="mr-2 pl-0"
                @click.stop="resetDatas(item.items)"
              >
                <span>{{ key }}</span>
                <v-chip
                  label
                  color="blue-grey"
                  small
                  text-color="white"
                  style="margin-right:-13px;"
                  class="ml-2"
                  disabled
                >
                  <span>{{ item.items.length }}</span>
                </v-chip>
              </v-chip>
            </template>
          </div>
          <!-- 分类列表 -->
          <div class="mt-1" v-if="showCategory">
            <template v-for="(item, key) in searchResult.categories">
              <v-chip
                :key="key"
                label
                color="grey darken-1"
                text-color="white"
                small
                class="mr-2 pl-0"
                @click.stop="resetDatas(item.items)"
              >
                <span>{{ key }}</span>
                <v-chip
                  label
                  color="grey"
                  small
                  text-color="white"
                  style="margin-right:-13px;"
                  class="ml-2"
                  disabled
                >
                  <span>{{ item.items.length }}</span>
                </v-chip>
              </v-chip>
            </template>
          </div>
        </v-flex>
        <!-- 操作按钮列表 -->
        <v-flex xs6>
          <div>
            <v-text-field
              v-model="filterKey"
              append-icon="search"
              :label="words.filterSearchResults"
              single-line
              hide-details
            ></v-text-field>
          </div>
          <v-layout row wrap align-center justify-end>
            <div>
              <v-switch
                color="success"
                v-model="checkBox"
                :label="words.showCheckbox"
                style="width: 100px;flex:none;float:right;"
              ></v-switch>
              <v-switch
                color="success"
                v-model="showCategory"
                :label="words.showCategory"
                style="width: 100px;flex:none;float:right;"
              ></v-switch>
            </div>

            <div>
              <!-- 复制下载链接 -->
              <v-btn
                :disabled="selected.length==0"
                color="success"
                small
                :title="words.copyToClipboardTip"
                @click="copySelectedToClipboard()"
              >
                <v-icon class="mr-2" small>file_copy</v-icon>
                {{words.copyToClipboard}} ({{selected.length}})
              </v-btn>
              <!-- 发送到下载服务器 -->
              <v-btn
                :disabled="selected.length==0"
                color="success"
                small
                :title="words.sendToClientTip"
                @click.stop="showAllContentMenus($event)"
              >
                <v-icon class="mr-2" small>cloud_download</v-icon>
                {{words.sendToClient}} ({{selected.length}})
              </v-btn>

              <!-- 文件发送进度 -->
              <v-progress-circular
                v-if="sending.count>0"
                :rotate="-90"
                :size="60"
                :width="10"
                :value="sending.progress"
                color="primary"
              >{{ sending.progress.toFixed(0) }}%</v-progress-circular>

              <v-btn
                :disabled="selected.length==0"
                @click="downloadSelected"
                color="success"
                small
                :title="words.save"
              >
                <v-icon class="mr-2" small>get_app</v-icon>
                {{words.download}} ({{selected.length}})
              </v-btn>
              <!-- 文件下载进度 -->
              <v-progress-circular
                v-if="downloading.count>0"
                :rotate="-90"
                :size="60"
                :width="10"
                :value="downloading.progress"
                color="primary"
              >{{ downloading.progress.toFixed(0) }}%</v-progress-circular>
            </div>
          </v-layout>
        </v-flex>
      </v-card-title>

      <!-- 数据表格 -->
      <v-data-table
        v-model="selected"
        :search="filterKey"
        :headers="headers"
        :items="datas"
        :pagination.sync="pagination"
        :loading="loading"
        item-key="link"
        :class="'torrent'+(fixedTable?' fixed-table fixed-header v-table__overflow':'')"
        :select-all="checkBox"
        :rows-per-page-items="options.rowsPerPageItems"
        @update:pagination="updatePagination"
      >
        <template slot="items" slot-scope="props">
          <td v-if="checkBox">
            <v-checkbox v-model="props.selected" primary hide-details></v-checkbox>
          </td>
          <!-- 站点 -->
          <td class="center">
            <v-avatar size="18">
              <img :src="props.item.site.icon">
            </v-avatar>
            <br>
            <span class="captionText">{{ props.item.site.name }}</span>
          </td>
          <!-- 标题 -->
          <td class="titleCell">
            <a
              :href="props.item.link"
              target="_blank"
              v-html="props.item.titleHTML"
              :title="props.item.title"
              rel="noopener noreferrer nofollow"
              class="subheading font-weight-medium"
            ></a>
            <div
              class="sub-title captionText"
              v-if="props.item.tags && props.item.tags.length || props.item.subTitle"
            >
              <span class="mr-1" v-if="props.item.tags && props.item.tags.length">
                <span
                  :class="['tag', `${tag.color}`]"
                  v-for="(tag, index) in props.item.tags"
                  :key="index"
                >{{ tag.name }}</span>
              </span>
              
              <span v-if="props.item.subTitle">{{props.item.subTitle}}</span>
            </div>
          </td>
          <!-- 分类 -->
          <td class="category center">
            <span
              v-if="props.item.category && !!props.item.category.name"
              :title="props.item.category.name"
              class="captionText"
            >{{ props.item.category.name }}</span>
            <br>
            <span class="captionText">&lt;{{ props.item.entryName }}&gt;</span>
          </td>
          <td class="size">{{props.item.size | formatSize}}</td>
          <!-- <td class="center">{{ props.item.comments }}</td> -->
          <td class="size">{{ props.item.seeders }}</td>
          <td class="size">{{ props.item.leechers }}</td>
          <td class="size">{{ props.item.completed }}</td>
          <!-- <td>{{ props.item.author }}</td> -->
          <td>{{ props.item.time | formatDate }}</td>
          <td>
            <v-icon
              small
              class="mr-2"
              @click="copyLinkToClipboard(props.item.url)"
              :title="words.copyToClipboardTip"
            >file_copy</v-icon>

            <!-- 服务端下载 -->
            <v-icon
              @click.stop="showSiteContentMenus(props.item, $event)"
              small
              class="mr-2"
              :title="words.sendToClient"
            >cloud_download</v-icon>

            <a
              :href="props.item.url"
              target="_blank"
              rel="noopener noreferrer nofollow"
              :title="words.save"
            >
              <v-icon small class="mr-2">get_app</v-icon>
            </a>
          </td>
        </template>
      </v-data-table>
    </v-card>

    <v-snackbar v-model="haveError" top :timeout="3000" color="error">{{ errorMsg }}</v-snackbar>
    <v-snackbar v-model="haveSuccess" bottom :timeout="3000" color="success">{{ successMsg }}</v-snackbar>
  </div>
</template>
<script lang="ts" src="./SearchTorrent.ts"></script>
<style lang="scss" src="./SearchTorrent.scss"></style>
<style lang="scss" src="./contextMenu.scss"></style>