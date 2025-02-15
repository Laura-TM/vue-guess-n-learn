<template>
  <main class="container px-8 mx-auto lg:px-4">
    <div id="commandLandscape" v-show="displayCommand">
      <h1 class="text-center">
        Please choose a character for the computer to guess
      </h1>
    </div>
    <div class="flex" id="objectlist">
      <ObjectList @messageFromChild="objectSelected" />
      <SelectedCharacter
        v-if="characterSelected"
        :yourSelection="yourSelection"
        :displayAskButton="displayAskButton"
        @messageFromChild="guess"
      />
      <div id="commandPortrait" v-show="displayCommand">
        <h1 class="text-center">
          Please choose a character for the computer to guess
        </h1>
      </div>
    </div>
  </main>
</template>

<style>
@media (orientation: landscape) {
  #container {
    margin-top: 1vh;
  }
  #commandLandscape h1 {
    font-size: 3vh;
    margin-top: 0.5vh;
    margin-bottom: 1vh;
  }
  #commandLandscape {
    height: 5vh;
  }
  #commandPortrait {
    display: none;
  }
  #objectlist {
    height: 92vh;
  }
}

@media (orientation: portrait) {
  #commandPortrait h1 {
    margin-top: 1vh;
    margin-bottom: 2vh;
  }
  #commandPortrait {
    height: 15vh;
    justify-content: center;
    display: flex;
    align-items: center;
  }
  #commandLandscape {
    display: none;
  }
  #objectlist {
    height: 99vh;
    flex-direction: column-reverse;
  }
}
</style>

<script lang="ts">
import { Options, Vue } from "vue-class-component";
import ObjectList from "../components/ObjectList.vue";
import SelectedCharacter from "../components/SelectedCharacter.vue";
import Swal from "sweetalert2";
import { IObject } from "../interfaces/IObject";
import GuessClass from "../classes/Guess";
import ObjectClass from "../classes/ObjectClass";
import { get, post } from "../helpers/http";
import router from "../router";

@Options({
  data: () => {
    return {
      yourSelection: "",
      characterSelected: false,
      displayAskButton: false,
      displayCommand: true,
    };
  },
  components: {
    ObjectList,
    SelectedCharacter,
  },
})
export default class ChooseObject extends Vue {
  protected yourSelection: string = "" as string;
  protected characterSelected: boolean = false as boolean;
  protected displayAskButton: boolean = false as boolean;
  protected displayCommand: boolean = true as boolean;

 async selectLanguage(languageCode: string): Promise<void> {
    const ok = await Swal.fire({
      title: "Language",
      text: "Changed language to " + languageCode
    });

    if (ok) {
      await get<IObject>("/api/learn-language/" + languageCode);
    }
  }

  async created(): Promise<void> {
    if (Object.keys(this.$route.params).length > 0) {
      let objectSelected = await get<IObject>("/api/select");

      this.yourSelection = ObjectClass.getImage(objectSelected);
      this.characterSelected = true;
      this.displayAskButton = true;
      this.displayCommand = false;
    }
  }

  async objectSelected(objects: Array<IObject>, index: number): Promise<void> {
    let name = objects[index].name;
    const data = { selection: name };

    const ok = await Swal.fire({
      title: "Your selection",
      text: "You have selected " + name,
      reverseButtons: true,
    });

    if (ok) {
      await post<any>("/api/select", data);
      this.yourSelection = ObjectClass.getImage(objects[index]);
      this.characterSelected = true;
      this.displayCommand = false;
      this.guess();
    }
  }

  async guess(sentence?: string, computerChoice?: string): Promise<void> {
    let question = await GuessClass.getComputerQuestion(
      sentence,
      computerChoice
    );
    let answer = await GuessClass.displayGuessDialog(question);

    if (question.choice == "") {
      router.push("game-over");
      return;
    }

    const data = { correct: answer, choice: question.choice };
    let response = await post<any>("/api/computer-guess", data);

    if (answer !== response.correct) {
      return this.guess(question.sentence, question.choice);
    }

    router.push("pick-attribute");
  }
}
</script>